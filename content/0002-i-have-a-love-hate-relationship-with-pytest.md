Title: I have a love hate relationship with pytest
Date: 2020-07-04 10:43
Category: Python

The community has spoken. If you're green fielding a python project, the best bet for your tests is using [pytest](https://docs.pytest.org/en/stable/).
Whether you need parallel test support, code coverage, or integration with some library you're using, you can most likely
find an existing pytest plugin. What would it be like to take the things I love from pytest and apply
them to [unittest](https://docs.python.org/3.8/library/unittest.html)? Full code can be found [here](https://github.com/massover/unittestplus).

## Function based tests

For a lot of my tests in pytest, I still end up using classes as they are an additional nice way to namespace your tests
on top of modules and packages. However, when there are only a few code paths to test, I find that function 
based tests are more readable and easier to maintain than classes.

I came across a [FunctionTestCase](https://docs.python.org/3/library/unittest.html#unittest.FunctionTestCase)
class that exists in the stdlib. However, this test case is intended for legacy code, and they are [blocked from
test default discovery](https://bugs.python.org/issue22680)

We can build a decorator on top of a function to leverage the existing `unittest` framework. This will make the
function discoverable and we can use the existing test case assertions.

**Note**

All of the tests will be around a dataclass for a Dog class.

```python
@dataclass
class Dog:
    name: str
    age: int = 0

def testcase(func):
    func_name = func.__name__
    if func_name.startswith('test') or func_name.endswith('test'):
        test_func_name = func_name
    else:
        test_func_name = f"test_{func.__name__}"
    testcase_name = f"TestCase__{func.__name__}"

    def method(self, *args, **kwargs):
        return func(self, *args, **kwargs)
    return type(testcase_name, (unittest.TestCase, ), {
        func_name: method
    })

@testcase
def test_dog_age(test):
    dog = Dog(name='Bruce', age=8)
    test.assertEqual(dog.age, 8)
```

```bash
python3 -m unittest discover -v 
test_dog_age (unittestplus.TestCase__test_dog_age) ... ok

----------------------------------------------------------------------
Ran 1 test in 0.000s

OK
```

## Parametrization

[Parametrization](https://docs.pytest.org/en/stable/parametrize.html) in pytest is great. The stdlib already has 
[subTests](https://docs.python.org/3/library/unittest.html#unittest.TestCase.subTest). A thin decorator wrapper around subTest
can make for some similar syntactical sugar. Personally, I prefer the decorator to the context manager.

```python
def parametrize(iterable):
    def inner(method):
        @functools.wraps(method)
        def wrapped(instance, *args, **kwargs):
            for msg_params in iterable:
                if isinstance(msg_params, dict):
                    msg = ''
                    params = msg_params
                else:
                    msg = msg_params[0]
                    params = msg_params[1]
                with instance.subTest(msg=msg, **params):
                    return method(instance, *args, **params, **kwargs)
        return wrapped
    return inner


class TestDog(unittest.TestCase):
    @parametrize([
        ('Name is Bruce', {'name': 'Bruce'}),
        ('Name is Peneloope', {'name': 'Penelope'}),
    ])
    def test_dog_name_msg_and_params(self, name):
        dog = Dog(name=name)
        self.assertEqual(dog.name, name + "fail")
```

```bash
python3 -m unittest discover -v 
test_dog_name_msg_and_params (test_example.TestDog) ... 
======================================================================
FAIL: test_dog_name_msg_and_params (test_example.TestDog) [Name is Bruce] (name='Bruce')
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/Users/joshmassover/Development/unittestplus/unittestplus.py", line 19, in wrapped
    return method(instance, *args, **params, **kwargs)
  File "/Users/joshmassover/Development/unittestplus/test_example.py", line 46, in test_dog_name_msg_and_params
    self.assertEqual(dog.name, name + 'fail')
AssertionError: 'Bruce' != 'Brucefail'
- Bruce
+ Brucefail
?      ++++


======================================================================
FAIL: test_dog_name_msg_and_params (test_example.TestDog) [Name is Peneloope] (name='Penelope')
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/Users/joshmassover/Development/unittestplus/unittestplus.py", line 19, in wrapped
    return method(instance, *args, **params, **kwargs)
  File "/Users/joshmassover/Development/unittestplus/test_example.py", line 46, in test_dog_name_msg_and_params
    self.assertEqual(dog.name, name + 'fail')
AssertionError: 'Penelope' != 'Penelopefail'
- Penelope
+ Penelopefail
?         ++++


----------------------------------------------------------------------
Ran 1 test in 0.001s

FAILED (failures=2)

```

## Fixtures

Fixtures are a neat way to reuse code across tests. I didn't think a direct translation of pytest fixtures is appropriate
when working mostly in TestCase classes. Instead, [descriptor](https://docs.python.org/3/howto/descriptor.html) based 
declarative classes seems to fit the style. Borrowing the term "scope" from pytest, I introduce a [TestCasePlus](https://github.com/massover/unittestplus/blob/master/unittestplus.py#L77) 
TestCase that handles the fixture initialization for both `setUp` and `setUpClass`.

```python
class Fixture:
    FUNC_SCOPE = 'func'
    CLASS_SCOPE = 'class'
    scope = None

    def __init__(self, fn=None, scope=None):
        scope = scope or self.scope or self.FUNC_SCOPE
        super().__init__()
        assert scope in [self.CLASS_SCOPE, self.FUNC_SCOPE], \
            f"Invalid scope choice: {scope}. "
        self.fn = fn
        self.scope = scope
        self._fixture = None

    def __call__(self):
        return self.fn()

    def __get__(self, instance, owner):
        if self.name in instance._class_fixtures_cache:
            self._fixture = instance._class_fixtures_cache[self.name]
        elif self._fixture is None:
            self._fixture = self()
        return self._fixture

    def __set__(self, instance, value):
        self._fixture = value

    def __set_name__(self, owner, name):
        self.name = name


def fixture(scope=Fixture.FUNC_SCOPE):
    default_scope = scope

    def inner(func):
        @functools.wraps(func)
        def wrapped(scope=None):
            scope = scope or default_scope
            return Fixture(func, scope=scope)
        return wrapped

    return inner

@fixture()
def penelope_fixture():
    return Dog(name='Penelope', age=10)

class TestFixtureDecorator(TestCasePlus):
    penelope = penelope_fixture()
    penelope_class = penelope_fixture(scope="class")

    def test_penelope_fixture(self):
        self.assertEqual(self.penelope.name, penelope_fixture()().name)
        # test the side effect that function based fixtures are recreated on setup
        # this relies on the named order of the test cases.
        self.penelope.name = 'lol'

    def test_penelope_fixture_is_recreated(self):
        self.assertEqual(self.penelope.name, penelope_fixture()().name)

    def test_penelope_class(self):
        self.assertEqual(self.penelope_class.name, penelope_fixture(scope="class")().name)
        # test the side effect that function based fixtures are recreated on setup
        # this relies on the named order of the test cases.
        self.penelope_class.name = 'lol'

    def test_penelope_class_was_modified(self):
        self.assertEqual(self.penelope_class.name, 'lol')
```

```bash
python3 -m unittest discover -v 
test_penelope_class (test_example.TestFixtureDecorator) ... ok
test_penelope_class_was_modified (test_example.TestFixtureDecorator) ... ok
test_penelope_fixture (test_example.TestFixtureDecorator) ... ok
test_penelope_fixture_is_recreated (test_example.TestFixtureDecorator) ... ok

----------------------------------------------------------------------
Ran 4 tests in 0.000s

OK
```

This code only scratches the surface of what pytest implements. If you go further, you'll find the stdlib hard to extend
for things like plugins, cli, complex nested and session based fixtures, assertion re-writing, to name a few. Would you
ever choose unittest over pytest and why? 
