# How the Atheris Python Fuzzer Works - Google Security Blog(번역)

[원본 링크](https://security.googleblog.com/2020/12/how-atheris-python-fuzzer-works.html)

Posted by lan Eldred Pudney, Google Information Security

# 서론

[Fuzzing](https://en.wikipedia.org/wiki/Fuzzing) 이라는 단어를 

# 본문

<br>

금요일에 저희는 [Atheris Python fuzzing engine](Atheris Python fuzzing engine)을 오픈소스로 공개했다는 점을 [공지](https://opensource.googleblog.com/2020/12/announcing-atheris-python-fuzzer.html)했습니다.<br>
이 포스트에서는 퍼징 엔진이 나오기 까지에 대한 배경과 특징과 작동하는 방법에 대해서 이어나가겠습니다.

## 프로젝트의 기원(Origin)

2013년부터 지금까지 매년 구글 사내에서 "Fuzzit"이라고, fuzzer(이하 퍼저)를 만드는 대회를 열었습니다.<br>
2019년 10월이 되면서, 구글에서 사용하고 있는 C/C++로 작성된 오픈소스 코드 대부분에 사용할 수 있는 퍼저를 만든 상황이었습니다.<br>

So for that Fuzzit, the author of this post wrote a Python fuzzing engine based on libFuzzer.
https://llvm.org/docs/LibFuzzer.html

파이썬 퍼징 엔진을 만들고 난 뒤로, 구글에선 50개가 넘는 파이썬 퍼저가 만들어졌고, 수많은 유지보수가 이루어졌습니다.


Originally, this fuzzing engine could only fuzz native extensions, as it did not support Python coverage. But over time, the fuzzer morphed into Atheris, a high-performance fuzzing engine that supports both native and pure-Python fuzzing

## 알고가면 좋은 사전지식

Atheris는 coverage-guided 퍼저입니다.

Atheris를 사용하기 전에 `atheris.Setup()`을 통해 테스트 케이스를 입력하는 '엔트리 포인트(entry point)'을 설정해야합니다.<br>
Atheris는 엔트리 포인트를 계속해서 시스템/논리적 오류가 일어나도록 다른 값으로 테스트합니다.<br>
테스트를 계속 진행하면서 Atheris는 프로그램을 계속 모니터링하면서 오류가 더 잘 일어나게끔하는 특정 값을 계속 찾아나갑니다.<br>
단순히 다른 값을 가진 테스트 케이스로 테스트를 하지 않기 때문에 버그를 발생하는 값을 찾는데 더 효율적입니다.

```py
import atheris
import sys

def TestOneInput(data): # 엔트리 포인트
    if data == b'bad':
        raise RuntimeError('badness!')
        
atheris.Setup(sys.argv, TestOneInput)
atheris.Fuzz()
```

Atheris는 파이썬으로 제작되었으며, 대표적으로 사용하는 외부라이브러리는 C/C++로 제작된 [libFuzzer](https://llvm.org/docs/LibFuzzer.html)을 이용해 [코드 커버리지](https://ko.wikipedia.org/wiki/코드_커버리지)
를 확보하고 테스트 케이스를 생성합니다.<br>
`atheris.Setup()`을 통해 전달된 엔트리 포인트는 C++ 확장 코드로 래핑되어 libFuzzer와 호환이 가능하도록 설계되어 있습니다.<br>
libFuzzer에서 생성된 값은 파이썬에 맞는 타입으로 래핑되어 테스트를 수행할 수 있습니다.

## 파이썬 코드 커버리지

Atheris는 컴파일된 libFuzzer가 링크된 상태로 작성된 파이썬 프로그램입니다.<br>
Atheris를 시작할 때, Cpython으로부터 실행중인 함수를 추적할 [추적기(tracer)](https://nedbatchelder.com/text/trace-function.html)를 등록합니다.<br>
추적기를 등록함으로써 실행중인 함수의 줄 하나하나를  추적할 수 있습니다.

추적한 정보들은 코드 커버리지 정보를 생성하는 libFuzzer에게 전달해야합니다.<br>
C/C++ 대상으로 작성된 libFuzzer과 파이썬 사이에 발생하는 문제는, libFuzzer가 분석하는 코드는 컴파일 타임에 발생한 문제로 알고 있다는 점입니다.<br>

코드 커버리지의 주요 메커니즘은 

* 방문한 프로그램 카운터 세트(set)를 등록하는  `__sanitizer_cov_pcs_init`

* 기본 블럭(basic block)이 방문할 경우 횟수를 증가할지 여부를 나타내는 불값 배열을 등록하는 `__sanitizer_cov_8bit_counters_init`

입니다.

원칙상, 프로그램 카운터의 개수와 기본 블럭이 얼마나 있는지 알기 위해서, 두개의 메커니즘 모두 프로그램이 초기화를 할 때 작동해야합니다.<br>
하지만 파이썬은 컴파일 단계를 거치지 않기 때문에 의도대로 작동할 수 없었습니다.

동적으로 파이썬 코드가 실행 중에 임포트하거나 동작 도중에 코드를 생성하여 퍼징을 할 수 있지만, 퍼징이 시작할 정확한 시점은 정해지지 않은 문제가 있었습니다.<br>
다행히 libFuzzer는 런타임 도중 실행할 수 있었기 때문에, 파이썬의 컴파일 여부 상관 없이 동적으로 사용할 수 있었습니다.

`__sanitizer_cov_pcs_init`, `__sanitizer_cov_8bit_counters_init` 모두 공유된 라이브러리가 로드되면서  내부에 있는 생성자로부터 호출할 수 있었습니다.<br>

추적이 시작되면, Atheris는 *8bit\_counter*와 프로그램 카운터를 호출합니다.

Atheris가 파이썬 코드를 읽으면서 한 줄마다 각각 PC와 *8bit\_counter*를 그 줄에 할당합니다.

<!-- 여기서 PC는 뭘 말하는거지? 스레드? -->
Atheris allocates a PC and 8-bit counter to that line; Atheris will always report that line the same way from then on

Atheris가 할당할 PC와 *8bit\_counter*가 모자르면 공유된 라이브러리를 위에서 언급한 두 개의 함수를 호출하면서 위와 같은 과정을 반복합니다.<br>
메모리에 계속해서 올라오도록 하는 이유는 하나의 라이브러리가 너무 많은 코드를 담당하지 않도록 하기 위해서입니다.

## 왜 Python 3.8 버전 이상부터 사용할 수 있나요?

Atheris 문서 [README](https://github.com/google/atheris/blob/master/README.md)에 나와있듯이 사용자에게 파이썬 버전 3.8 이상을 권장하고 있습니다.<br>
파이썬 버전 3.8부터 **opcode tracing**이 추가됐기 때문입니다.

새로 추가된 기능을 이용해서 실행 도중에 코드의 몇번 째 줄이 몇번 방문됐는지, <br>
어떤 함수가 호출됐는지, 함수 인자에는 어떤 값이 들어왔는지, 등 모니터링 할 수 있기 때문입니다.<br>

이 기능을 이용하면 Atheris가 `if` 문을 찾는데 더 쉬워집니다.<br>
예를 들어, 불린 값을 비교하는 `COMPARE_OP` opcode를 확인 했을 때 Atheris는 값의 타입을 검사합니다.<br>
비교할 값이 `bytes` 또는 `Unicode`일 경우 Atheris는 `__sanitizer_weak_hook_memcmp`를 통해 비교 연산을 libFuzzer로 보고합니다.<br>
정수끼리 비교 연산을 하는 경우, Atheris는 정수 비교 연산을 보고하는 `__sanitizer_cov_trace_cmp8`을 이용해서 보고합니다.

최근 파이썬 버전에서는, 유니코드 문자열은 문자 배열에서 가장 큰 문자를 기준으로 1-byte, 2-byte, 4-byte 문자의 배열로 표현합니다.<br>
이와 같이 확정되지 않은 자료형 타입의 문제를 해결하기 위해 사용되는 방법은

1. 두 문자열 간에 문자 사이즈를 비교하고, 비교 연산을 한것을 `__sanitizer_cov_trace_cmp8`로 보고합니다.

2. 동일한 경우, `__sanitizer_weak_hook_memcmp`을 이용해 문자열 비교 연산을 보고합니다.

이런 방식으로 문자열을 비교하는 연산을 수행해왔고 이런 방식이 최선의 방식이라고 생각했지만,<br>
비교할 유니코드 문자열을 모두 utf-8로 변환한뒤 `__sanitizer_weak_hook_memcmp`을 사용하는 것이 더 빠른것으로 나타났습니다(...).<br>
변환하는 과정에서 오버헤드로 인한 성능저하가 우려되었지만, 더 빠르게 작동했습니다.

## Atheris 빌드하기

Atheris를 릴리스하는데 가장 노력을 쏟은 것은 구글 환경 밖에서 만드는 것이었습니다.<br>
구글에서 파이썬 프로젝트를 진행한다는 것은 사내 파이썬 인터프리터를 포함한 다양한 라이브러리까지 의존성이 발생할 수 밖에 없었습니다.<br>
사내 라이브러리 및 인터프리터 환경에 의존성을 갖는다는 것은 libFuzzer를 이용하는데 불편할 수 밖에 없었습니다.

구글의 영향 밖에서 만든다는 것은 쉽지 않은 일이었습니다.<br>
libFuzzer를 Atheris에 적용시키는 것부터 시작해서, 의존성이 없는 프로젝트를 만들기까지 많은 고난이 있었습니다.<br>
Atheris의 공유된 오브젝트에 libFuzzer와 링크를 잇는 것이 사용자 입장에서 최선의 경험으로 판단해 이렇게 시작했습니다.

하지만 이 방법도 결국 libFuzzer를 온전히 사용할 수 없었기 때문에 수정이 필요했습니다.<br>
대부분의 사용자들의 최신 Clang을 사용하지 않는다는 점과 Clang의 최신판이 모두 베포되기 까지 수년이 걸린다는 것을 알고 있었기 때문에<br>
Atheris에 사용하기 적절하게 업데이트 될 새로운 libFuzzer를 사용할 가능성은 적어보였습니다.<br>
버전 차이로 인한 혼란을 막기 위해서, Atheris를 설치할 경우 libFuzzer의 버전이 너무 낮은 경우 업데이트하도록 만들었습니다.<br>

Atheris의 [setup.py](https://github.com/google/atheris/blob/master/setup.py)는
사용자 컴퓨터에 있는 libFuzzer의 버전의 너무 낮은 경우, 복사본을 만든 뒤, 퍼저 [엔트리 포인트](https://github.com/llvm/llvm-project/blob/ca2888310b245d0532d989685a090ae373ee3f93/compiler-rt/lib/fuzzer/FuzzerDriver.cpp#L639)를
 표시한 뒤에 *LLVMFuzzerRunDriver*라는 이름으로 호출될 수 있도록 [래퍼(wrapper)](https://github.com/google/atheris/blob/master/setup_utils/fuzzer_run_driver_wrapper.cc)를 설치합니다.<br>
 libFuzzer이 비교적 최근에 업데이트 된 경우, [LLVMFuzzerRunDriver](https://github.com/llvm/llvm-project/blob/ca2888310b245d0532d989685a090ae373ee3f93/compiler-rt/lib/fuzzer/FuzzerDriver.cpp#L917)라는 이름으로 직접 사용합니다.

```
아래 내용 선요약 : 

Sanitizer가 심볼을 가져가기 때문에 코드가 길어져서 라이브러리를 또 로드하는 경우, 2번 이상, 앞서 바뀐 심볼에 접근 할 수 없어 제대로 작동할 수 없다는 이야기<br>
+
코드가 길어지는 경우 로드되는 libFuzzer 라이브러리가 위 문제 떄문에 제대로 동작할 수 없다는 것
```

진짜 문제는 native extensions를 sanitizers와 함께 실행했을 경우에 드러납니다.<br>
이론적으로 native extensions를 퍼징할 경우 단순히 `-fsanitize=fuzzer-no-link` 한줄과 함께 빌드하고, 실행하기전 Atheris가 먼저 로드가 되었는지만 확인하면 됩니다.<br>
이렇게 하면 Clang이 Atheris 내부에 정의된 libFuzzer 심볼을 포인팅하도록 만들기 때문에 경로(?) 문제를 방지할 수 있습니다.

모든 문제는 sanitizer로부터 시작됩니다.

Address Sanitizer(이하 ASan)과 같은 sanitizer를 Atheris와 함께 사용할 경우, sanitizer의 공유된 오브젝트를 `LD_PRELOAD`하는 것이 원칙입니다.<br>
하지만 ASan은 무조건 첫번째로 로드되는 것을 요구하기 때문에 먼저 로드 되어있거나, 실행 가능한 파일(파이썬 인터프리터)에 정적으로 링크 되어있어야합니다.<br>
ASan과 UBSan(?)은 libFuzzer과 겹치는 심볼이 많기 때문에, 코드가 길어질 경우 libFuzzer 라이브러리가 로드되었을 때 코드 커버리지에 대한 정보가<br>
이미 `LD_PRELOAD`된 심볼로 로드되기 때문에 libFuzzer가 제대로 작동할 수 없게됩니다.

이 방법을 해결할 수 있는 것은 libFuzzer를 파이썬 자체에서 직접 링크하는 것입니다.<br>
나중에 동적으로 메모리를 로드하는 것보다 실행하는 방법이 더 옳은 편이며, libFuzzer 심볼에 대해 문제를 없앨 수 있기 때문입니다.

이 문제에 대한 [문서](https://github.com/google/atheris/blob/master/using_sanitizers.md)가 있으며 CPython 3.8.6 버전에 적용할 수 있는 [스크립트](https://github.com/google/atheris/blob/master/third_party/build_modified_libfuzzer.sh)를 제공한 상태입니다.

