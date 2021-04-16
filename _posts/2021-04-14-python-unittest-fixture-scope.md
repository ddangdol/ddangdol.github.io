---
layout: posts
title: python unittest 의 fixture scope
---

## python testing tool ##
python 에는 널리 알려져 사용되는 여러 도구들이 이미 많습니다. 특히 pytest 는 좀 더 직관적인 assert 구문과 test 결과를 제공하며, 많은 plugin 을 보유하고 있습니다. 하지만 여러 도구들을 알아보기 전에 python testing에 기반이 되는 unittest 프레임워크의 개념과 사용법에 대해 알아보겠습니다.

오늘은 그 중 fixture의 종류와 사용범위에 대해 알아보겠습니다.

---
### Test Fixture ###
테스트 픽스쳐는 테스트를 수행 시 필요한 리소스 준비과 이를 정리하는 동작에 해당합니다. unittest 프레임워크에서는 fixture를 여러 scope에서 사용할 수 있도록 기능을 제공합니다. 아래에서 Test, Class, Module Scope를 위한 fixture 사용법에 대해 알아보겠습니다.

#### Test Scope ####
TestCase 의 각 테스트에서 사용될 fixture의 준비와 정리는 setUp()과 tearDown()을 사용하게 됩니다. 테스트마다 반복될 수 있는 fixture를 setUp() 메소드와 tearDown() 메소드를 사용해 분리할 수 있습니다. 테스트 프레임워크가 각 테스트 실행 전후에 두 메소드를 자동으로 호출합니다.
```python
import unittest


class RemainderTest(unittest.TestCase):
  def setUp(self) -> None:
    self.number = 2
    print('setUp')

  def tearDown(self) -> None:
    print('tearDown')

  def test_even(self):
    self.assertEqual(self.number % 2, 0)

  def test_odd(self):
    self.assertNotEqual(self.number % 2, 1)
```
위 테스트를 실행한 결과를 통해 각 테스트별로 매번 fixture가 생성되는 것을 알 수 있습니다.
```console
setUp
tearDown
setUp
tearDown
```
setUp() 메소드 작업에서 예외가 발생하면 tearDown 메소드는 호출되지 않습니다.
```python
class RemainderTest(unittest.TestCase):
    def setUp(self) -> None:
        self.number = 2
        print('setUp')
        raise Exception()

    def tearDown(self) -> None:
        print('tearDown')

    def test_even(self):
        self.assertEqual(self.number % 2, 0)

    def test_odd(self):
        self.assertNotEqual(self.number % 2, 1)
```
```console
setUp
Error
Traceback (most recent call last):
...
setUp
Error
Traceback (most recent call last):
...
```
setUp 메소드 실행이 성공했다면 테스트 실패 여부과 관계없이 tearDown 메소드가 호출됩니다.

테스트 중에 사용된 자원을 정리하기 위해 tearDown 이후에 불리는 함수를 추가할 수 있습니다. 추가된 순서의 반대 순서(LIFO)로 불리게 됩니다. 함수가 추가될 때 addCleanup에 같이 전달된 위치 인자나 키워드 인자와 함께 호출됩니다.
```python
def cleanUp():
    print('cleanUp')


class RemainderTest(unittest.TestCase):
    def setUp(self) -> None:
        self.number = 2
        print('setUp')
        self.addCleanup(cleanUp)

    def tearDown(self) -> None:
        print('tearDown')

    def test_even(self):
        self.assertEqual(self.number % 2, 0)
```
```console
setUp
tearDown
cleanUp
```
또한 setUp 메소드에서 예외가 발생하면 tearDown 메소드가 호출되지 않아도 doCleanups 메소드가 실행됩니다.
```python
def cleanUp():
    print('cleanUp')


class RemainderTest(unittest.TestCase):
    def setUp(self) -> None:
        self.number = 2
        print('setUp')
        self.addCleanup(cleanUp)
        raise Exception()

    def tearDown(self) -> None:
        print('tearDown')

    def test_even(self):
        self.assertEqual(self.number % 2, 0)
```
```console
setUp
cleanUp

Error
Traceback (most recent call last):
```
doCleanUp 메소드를 직접 호출하여 addCleanUp에 추가된 함수들을 tearDown 호출 이전에 정리할 수도 있습니다.
```python
def cleanUp():
    print('cleanUp')


class RemainderTest(unittest.TestCase):
    def setUp(self) -> None:
        self.number = 2
        print('setUp')
        self.addCleanup(cleanUp)

    def tearDown(self) -> None:
        print('tearDown')

    def test_even(self):
        self.assertEqual(self.number % 2, 0)
        self.doCleanups()
```
```console
setUp
cleanUp
tearDown
```

#### Class Scope ####
Test Class 범위에 fixture는 unittest.TestSuite에 의해 관리되는 setUpClass와 tearDownClass가 있습니다. 해당 fixture는 Test Class의 모든 테스트와 공유되는 점을 주의해야 합니다. 아래와 같이 특정 테스트에서 str_list fixture의 수정이 일어나게 되면 이후 테스트에 영향을 미치게 됩니다.
```python
class JoinTest(unittest.TestCase):
    @classmethod
    def setUpClass(cls) -> None:
        cls.str_list = ['foo', 'bar']
        print('setUpClass')

    @classmethod
    def tearDownClass(cls) -> None:
        print('tearDownClass')

    def test_join_with_colon(self):
        expected = 'foo:bar'
        self.assertEqual(':'.join(self.str_list), expected)
        self.str_list.append('baz')

    def test_join_with_comma(self):
        expected = 'foo,bar'
        self.assertEqual(','.join(self.str_list), expected)
        self.str_list.append('baz')
```
setUpClass 메소드 작업에서 예외가 발생하면 tearDownClass 메소드는 호출되지 않습니다.

test scope와 마찬가지로 cleanUp을 위한 addClassCleanup 메소드를 제공합니다. setUpClass 메소드에서 예외가 발생하면 tearDownClass 메소드가 호출되지 않아도 doClassCleanups 메소드가 실행됩니다.
```python
def classCleanUp():
    print('classCleanUp')


class JoinTest(unittest.TestCase):
    @classmethod
    def setUpClass(cls) -> None:
        cls.str_list = ['foo', 'bar']
        print('setUpClass')
        cls.addClassCleanup(classCleanUp)

    @classmethod
    def tearDownClass(cls) -> None:
        print('tearDownClass')

    def test_join_with_colon(self):
        expected = 'foo:bar'
        self.assertEqual(':'.join(self.str_list), expected)
```
addClassCleanup에 추가된 함수들을 tearDownClass 이전에 정리하고 싶다면 doClassCleanups 메소드를 사용해 미리 정리할 수 있습니다.

#### Module Scope ####
module scope 에서는 setUpModule과 tearDownModule을 제공하며 함수로 작성되어야 합니다. 마찬가지로 setUpModule 실행 중 예외가 발생하면 tearDownModule 함수는 호출되지 않습니다.
```python
def setUpModule():
    print('setUpModule')


def tearDownModule():
    print('tearDownModule')


class JoinTest(unittest.TestCase):
    def test_join_with_colon(self):
        expected = 'foo:bar'
        self.assertEqual(':'.join(['foo', 'bar']), expected)


class RemainderTest(unittest.TestCase):
    def test_even(self):
        self.assertEqual(2 % 2, 0)
```
```console
setUpModule
tearDownModule
```
module scope에 cleanUp을 위해 unittest.addModuleCleanup 함수를 제공합니다. tearDownModule 함수가 호출된 이후 unittest.case.doModuleCleanups 함수를 통해 추가된 cleanUp 함수들이 호출됩니다.
```python
def setUpModule():
    print('setUpModule')


def tearDownModule():
    print('tearDownModule')


def moduleCleanUp():
    print('moduleCleanUp')


class JoinTest(unittest.TestCase):
    def test_join_with_colon(self):
        expected = 'foo:bar'
        self.assertEqual(':'.join(['foo', 'bar']), expected)
        unittest.addModuleCleanup(moduleCleanUp)
```
```console
setUpModule
tearDownModule
moduleCleanUp
```
tearDownModule이 호출되기 전에 추가된 정리 함수들을 호출하고 싶다면 unittest.case.doModuleCleanups 함수를 직접 호출합니다.
```python
def setUpModule():
    print('setUpModule')


def tearDownModule():
    print('tearDownModule')


def moduleCleanUp():
    print('moduleCleanUp')


unittest.addModuleCleanup(moduleCleanUp)


class JoinTest(unittest.TestCase):
    def test_join_with_colon(self):
        expected = 'foo:bar'
        self.assertEqual(':'.join(['foo', 'bar']), expected)
        unittest.case.doModuleCleanups()


class RemainderTest(unittest.TestCase):
    def test_even(self):
        self.assertEqual(2 % 2, 0)
```
```console
setUpModule
moduleCleanUp
tearDownModule
```

#### 전체적인 fixture scope 흐름 ####
```python
def setUpModule():
    print('setUpModule')


def tearDownModule():
    print('tearDownModule')


def cleanUp():
    print('cleanUp')


def classCleanUp():
    print('classCleanUp')


def moduleCleanUp():
    print('moduleCleanUp')


unittest.addModuleCleanup(moduleCleanUp)


class JoinTest(unittest.TestCase):
    def setUp(self) -> None:
        print('setUp')
        self.addCleanup(cleanUp)

    def tearDown(self) -> None:
        print('tearDown')

    @classmethod
    def setUpClass(cls) -> None:
        print('setUpClass')
        cls.addClassCleanup(classCleanUp)

    @classmethod
    def tearDownClass(cls) -> None:
        print('tearDownClass')

    def test_join_with_colon(self):
        expected = 'foo:bar'
        self.assertEqual(':'.join(['foo', 'bar']), expected)
```
위에서 언급된 fixture scope 를 모두 포함한 결과를 보면 호출 순서는 아래와 같습니다.
```console
setUpModule
setUpClass
setUp
tearDown
cleanUp
tearDownClass
classCleanUp
tearDownModule
moduleCleanUp
```
### 정리 ###
python unittest를 사용해 테스트 작성 시 fixture 별 scope와 호출 시점과 조건을 잘 파악해야 합니다. 여러 scope의 fixture를 잘 활용하면 중복을 해결하고 테스트 성능에 도움이 되지만 잘못 사용할 경우 의도와 다른 테스트 결과가 출력될 수 있음에 주의하며 사용하는 것이 좋습니다.

---
### 참고 ###
* https://docs.python.org/ko/3/library/unittest.html

