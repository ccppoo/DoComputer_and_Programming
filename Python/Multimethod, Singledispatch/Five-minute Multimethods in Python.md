# Five-minute Multimethods in Python - Guido van Rossum

원본  : [Five-minute Multimethods in Python](https://www.artima.com/weblogs/viewpost.jsp?thread=101605)

이 글은 귀도 반 로썸이 멀티 메서드(multimethod)에 대해서 작성한 글입니다.

### 주의

멀티메서드는 파이썬 내장 모듈 `functools.singledispatch`에 존재하니 아래 예시의 모듈 대신 `singledispatch`를 사용하세요!<br>

## 목차

1. 번역

2. 데코레이터 패턴을 사용하지 않을 경우는요?

3. \_\_lastreg\_\_는 어디에 쓰이는 거에요?

4. 본문 내 사용된 코드 최종본

## 번역

역주 : 예시 코드의 최종본은 맨 아래에 있습니다!

#### 선요약

저는 평소 멀티 메서드(multimethod)에 대해서 쓸 일이 별로 없을거라고 생각했습니다.<br>
비록 아직도 그렇게 생각하지만, 한번 다뤄봤으면 해서 간단한 멀티 메서드를 파이썬으로 구현해봤습니다.<br>
기본적인 지식은 어느정도 요구하는 수준이며, 독자들이 직접 스스로 응용해볼 수 있는 여지를 남겨뒀습니다.

------

멀티 메서드란 무엇일까요?

제가 멀티 메서드에 대해서 이해하는 과정을 통해 얻은 정의에 대해서 말씀드리겠습니다.<br>
'하나의 함수에 인자의 타입에 따라 분류되는 여러개의 버전이 있는 것'입니다.

어떤 사람들은 이보다 더 세분화해서 인자의 **값**에 따라  다른 버전의 함수를 만드는 것또한 포함된다고 주장하지만,<br>
이 글에서는 그정도까지 깊게 다루지 않습니다.
(역주 : 값에 따라 분류하는 패턴 중에서 계약에 의한 프로그래밍-Design by Contract을 참고하세요)

<br>

간단한 예시를 들어보겠습니다.

한 타입에 두가지 인자를 받는 함수가 있다고 합시다.<br>
다음과 같이 사용할 수 있습니다:

```python
def foo(a, b):
    if isinstace(a, int) and isinstance(b, int):
        # ... code
    elif isinstace(a, float) and isinstance(b, float):
        # ... code
    elif isinstace(a, str) and isinstance(b, str):
        # ... code
    else:
        raise TypeError(f'unsupported argument types {type(a)}, {type(b)}')
```

이런 패턴은 유지보수하기 힘들뿐더러 함수를 읽는 사람조차 버겁습니다.<br>
(객체지향적이지 않습니다, 메직 메서드도 이름이 객체지향적인 것 같지만 실제로는 아닌것 처럼요).<br>

데코레이터를 이용한 멀티 메서드 디스패치(dispatch)를 구현하면 다음과 같습니다.

```python
from mm import multimethod

@multimethod(int, int)
def foo(a, b):
    # ... code for two ints...
    
@multimethod(float, float)
def foo(a, b):
    # ... code for two floats...
    
@multimethod(str, str)
def foo(a, b):
    # ... code for two strings...
```

이 글은 계속해서 어떻게 멀티메서드 데코레이터를 어떻게 정의할 수 있는지에 대해 서술할 것입니다.

멀티메서드는 생각보다 간단하게 구현할 수 있습니다.<br>
글로벌 레지스터리에 함수 이름(예 : 'foo')를 인덱스에 저장한 뒤에, 지정한 타입 튜플에 해당하는 함수를 호출하면 됩니다.

```python
# This is in the 'mm' module

registry = {}

class MultiMethod(object):

    def __init__(self, name):
        self.name = name
        self.typemap = {}

    def __call__(self, *args):
        types = tuple(arg.__class__ for arg in args) # generator expression
        function = self.typemap.get(types)
        if function is None:
            raise TypeError('No match')
        return function(*args)

    def register(self, types, function):
        if types in self.typemap:
            raise TypeError('duplicate registration')
        self.typemap[types] = function
```

지금까지 내용을 이해하셨으리라고 믿습니다.<br>
'class'와 'type'을 동의어처럼 사용했으니 그 부분에 대해서 이해해주시길 바랍니다.<br>
(역주 : 아래에서는 타입(type)으로 통일해서 작성했습니다)

* `__init__` 생성자는 인자로 주어진 이름(문자열)으로 MultiMethod 객체와 비어있는 typemap(인자 타입 딕셔너리)을 만듭니다.

* `__call__` 메서드는 MultiMethod 객체가 호출가능한(callable) 객체로 만듭니다.<br>
    MultiMethod 객체가 호출되면 주어진 인자의 타입에 해당되는 실제 함수를 참조합니다.<br>
    이 메서드에서 생략된 부분은 정확하지 않은 타입이 주어진 경우에 어떻게 대응할지에 대해서 구현한 부분입니다.<br>
    실제 멀티메서드 구현에서는 정의하지 않은 타입이 인자로 주어진 경우, 그 타입의 피상속(super) 타입을 찾는다는지 유연한 대처를 필요로 하는 구현부입니다.<br>
    이 부분의 경우 독자가 스스로 해결해보도록 남겨뒀습니다.
    
* `register(self, types, function)` 메서드는 새로운 인자 타입에 매핑되는 함수를 추가합니다.<br>
    *types*에는 함수(*function*)에 상응하는 인자의 타입들의 튜플(e.g. `(int, int)`)을 요구하고 *function*은 호출될 함수를 요구합니다.

`@multimethod` 데코레이터가 MultiMethod 객체를 반환하고,<br>
사용자가 멀티메서드를 호출했을 때 `register()` 메서드를 통해서 함수가 호출되는 것까지의 과정이 이루어집니다.

역주 : 위에서 만든 코드에 이어서 사용 예시를 보여드리겠습니다

```python
# mm 모듈(MultiMethod 클래스가 정의된)에 이어 쓰면 됩니다
if __name__ == '__main__':
    doSomething = MultiMethod('doSomething')
    
    # 함수의 이름과 상관없이 단지 참조를 하기 때문에 함수의 이름은 상관없습니다
    def aaaa(arg1, arg2):
        print(f'I got String \'{arg1}\' and \'{arg1}\'')
    
    # callable 한 모든 것이면 뭐든지 가능합니다, 람다함수도 말이죠
    doSomething.register((int, int), lambda a, b : a+b)
    
    doSomething.register((str, str), aaaa)
    
    # 1. (int, int)
    print(doSomething(5,10))    # 15

    # 2. (str, str)
    doSomething('hello','ccppoo')   # I got String 'hello' and 'hello'
```

MultiMethod 생성자를 직접 호출하지 않고 데코레이터 형태로 사용하기 위해서 계속 만들어보겠습니다.

```python
# mm 모듈(MultiMethod 클래스가 정의된)에 이어 쓰면 됩니다

def multimethod(*types):
    
    def register(function):    
        name = function.__name__
        mm = registry.get(name)
        if mm is None:
            mm = registry[name] = MultiMethod(name)
        mm.register(types, function)
        return mm
    
    return register
```

포지셔널 인자만 사용할 수 있다는 점을 감안한다면 사용할만 합니다.

<br>

기본값을 갖는 함수를 작성한다면 이렇게 작성해야한다고 생각할 수 있습니다.

```python
@multimethod(int, int):
def foo(a, b=10):
    # ,,,
```

하지만, 이렇게 작성해야합니다

```python
@multimethod(int, int)
def foo(a, b):
    # ...
    
@multimethod(int)
def foo(a):
    return foo(a, 10)   # 위에 있는 foo(a, 10)를 호출합니다
```

------

여기서 더 나아가보겠습니다

다양한 타입에 적용되는 구현을 하고 싶은 마음이 생겼다고 칩시다.<br>
`@multimethod` 데코레이터가 아래처럼 쌓이면 어떨지 생각해봅시다.

```python
@multimethod(int, int)
@multimethod(int)
def foo(a, b=10):
    # ...
```

이미 작성한 코드를 살짝만 바꾸면 가능합니다 (thread-safe하지 않지만, 딱히 상관 없습니다. 이 모든 과정이 import 시간에 이루어지기 때문입니다)

```python
def multimethod(*types):
    
    def register(function):
        function = getattr(function, '__lastreg__', function)
        name = function.__name__
        mm = registry.get(name)

        if mm is None:
            mm = registry[name] = MultiMethod(name)
            
        mm.register(types, function)
        mm.__lastreg__ = function
        return mm
        
    return register
```

> 역주 : `__lastreg__`은 최근 등록한 멀티메서드를 참조하는 것으로, 

4번 째 줄, `getattr( ... )`는 생소한 표현일 수 있습니다.<br>

```python
if hasattr(function, '__lastreg__'):
    function = function.__lastreg__
else:
    function = function # '아님 말고~'라고 한 것과 같습니다
```

위와 같은 표현이며, `getattr(x, 'y', z)`라고 하면 `x.y`가 `None`이 아닐 경우 `z`를 반환하라는 의미입니다.

<br>

더 정적인 프로그래밍 언어에서 `__lastreg__`의 속성에 대해서 타입 선언(혹은 Early Binding이라고 이해할 수 있죠)을 해야하지만, 파이썬은 그럴 필요가 없습니다.<br>

(`__lastreg__`라고 이름을 지은 것 자체에 의구심을 가지고 있을 텐데, 다른 함수로부터 참고를 막으려는 의도가 있습니다.<br>
`__xxx__` 형태의 이름을 가진 함수 속성을 사용하는 것은 같은 이름의 속성이 덮어 쓰이지(overwrite) 않도록 한 것입니다.)

<br>

## 데코레이터 패턴을 사용하지 않을 경우는요?

```python
from multimethod import multimethod

a = multimethod(int)(lambda  x : print(f'I got int {x}'))

a.register((str, ), lambda x : print(f'I got str {x}'))

a.register((float, ), lambda x : print(f'I got float {x}'))

a(10)   # I got int 10

a('asdasd') # I got str asdasd

a(3.0)  # I got float 3.0
```

데코레이터 또한 결국 함수(정확히는 callable 인스턴스)를 **반환**하는 함수입니다.

고로 우리가 호출할 대상, 즉 함수라고 생각하고 쓸 대상(여기서 'a')에 값을 대입하면됩니다.

다만, 처음 멀티메서드를 대입한 후, 반환된 것은 `multimethod` 인스턴스인 것을 인지하고 사용하셔야 합니다.<br>
직접 `register()`메서드를 사용한 것처럼요.

<br>

## \_\_lastreg\_\_는 어디에 쓰이는 거에요?

우선 파이썬 함수 인스턴스의 기본 속성은 아닙니다.<br>
구글에 검색해도 5페이지 밖에 안나오는 희귀한 단어 그 자체입니다. 즉 귀도가(Guido) 글을 작성하면서 만들어낸 속성이죠.<br>
이름도 던더(dunder, double underbar)로 감쌌으니 그 의도 자체를 파악하셨을 겁니다.

lastreg는 **last register**의 줄임말로 `register()`메서드에 인자로 들어온 인자(*function*)이 일반 함수가 아닌

**이미 멀티메서드에 등록된 함수**일 경우, 인자로 들어온 함수의 멀티 메서드가 담겨져 있는 레지스터에 추가로 등록하도록

중복을 방지하는 플래그(Flag)와 같은 존재입니다.

다만 MultiMethod 인스턴스를 참조하는 것이 아닌 MultiMethod가 처음 호출(call) 되었을 때 시작한 함수(아래에선 `def k(x)`)가 참조되어 있습니다.

```python
from multimethod import multimethod

def k(x):
    print(f'I got int {x}')
    
def kk(x):
    print(f'I got str {x}')
    
def kkk(x):
    print(f'I got float {x}')

a = multimethod(int)(k)

a.register((str, ), kk)

a.register((float, ), kkk)

bb = multimethod(int, int)(a)
# 이때 a의 MultiMethod 인스턴스에는 __lastreg__가 있으니 bb 또한 'k'라는 이름의 함수의 멀티메서드 중 하나로 취급받습니다
```

<br>

## 본문 내 사용된 코드 최종본

```python
registry = {}

class MultiMethod(object):

    def __init__(self, name):
        self.name = name
        self.typemap = {}

    def __call__(self, *args):
        types = tuple(arg.__class__ for arg in args) # generator expression
        function = self.typemap.get(types)
        if function is None:
            raise TypeError('No match')
        return function(*args)

    def register(self, types, function):
        if types in self.typemap:
            raise TypeError('duplicate registration')
        self.typemap[types] = function

def multimethod(*types):
    
    def register(function):
        function = getattr(function, '__lastreg__', function)
        name = function.__name__
        mm = registry.get(name)

        if mm is None:
            mm = registry[name] = MultiMethod(name)
            
        mm.register(types, function)
        mm.__lastreg__ = function
        return mm
        
    return register
        
if __name__ == '__main__':
    @multimethod(int, int)
    def a(a, b):
        print(f'I got int {a} and int {b}')

    @multimethod(str, str)
    def a(a, b):
        print(f'I got str \'{a}\' and str \'{b}\'')
        
    a(10, 20)
    
    a('hello', 'ccppoo')
```

