Schema validation just got Pythonic
===============================================================================

**schema** is a library for validating Python data structures, such as those
obtained from config-files, forms, external services or command-line
parsing, converted from JSON/YAML (or something else) to Python data-types.

Example
-------------------------------------------------------------------------------

Here is a quick example to get a feeling of **schema**, validating a list of
entries with personal iformation:

```python
>>> from schema import Schema, And, Use, Optional

>>> schema = Schema([{'name': And(str, len),
...                   'age':  And(Use(int), lambda n: 18 <= n <= 99),
...                   Optional('sex'): And(str, Use(str.lower),
...                                        lambda s: s in ('male', 'female'))}])

>>> data = [{'name': 'Sue', 'age': '28', 'sex': 'FEMALE'},
...         {'name': 'Sam', 'age': '42'},
...         {'name': 'Sacha', 'age': '20', 'sex': 'Male'}]

>>> sue, sam, sacha = schema.validate(data)
>>> sue['age']
28
>>> sue['sex']
'female'
>>> sam['age']
42
>>> sacha['sex']
'male'

```

If data is valid, `Schema.validate` will return the validated data
(optionally converted with `Use` calls, see below).

If data is invalid, `Schema` will raise `SchemaError` exception.


Installation
-------------------------------------------------------------------------------

Use [pip](http://pip-installer.org) or easy_install:

    pip install schema==0.1.1

Alternatively, you can just drop `schema.py` file into your project--it is
self-contained.

- **schema** is tested with Python 2.6, 2.7
- **schema** follows [semantic versioning](http://semver.org).

How `Schema` validates data
-------------------------------------------------------------------------------

### Types

If `Schema(...)` encounteres a type (such as `int`, `str`, `object`, etc),
it will check if correspoinding piece of data is instance of that type,
otherwise it will exit with error;

```python
>>> from schema import Schema

>>> Schema(int).validate(123)
123

>>> Schema(int).validate('123')
Traceback (most recent call last):
...
SchemaError: '123' should be instance of <type 'int'>

>>> Schema(object).validate('hai')
'hai'

```

### Callables

If `Schema(...)` encounteres a callable (function, class, of object with
`__call__` method) it will call it, and if return value evaluates to
`True` it will continue validating, else -- it will exit with error.

```python
>>> import os

>>> Schema(os.path.exists).validate('./')
'./'

>>> Schema(os.path.exists).validate('./non-existent/')
Traceback (most recent call last):
...
SchemaError: exists('./non-existent/') should evalutate to True

>>> Schema(lambda n: n > 0).validate(123)
123

>>> Schema(lambda n: n > 0).validate(-12)
Traceback (most recent call last):
...
SchemaError: <lambda>(-12) should evalutate to True

```

### "Validatables"

If `Schema(...)` encounteres an object with method `validate` it will run this
method on corresponding data as `data = smth.validate(data)`. This method may
raise `SchemaError` exit-exception, which will tell `Schema` that that piece
of data is invalid, otherwise -- it will continue to validate.

As example, you can use `Use` for creating such objects. `Use` helps to use
a function or type to convert a value while validating it:

```python
>>> from schema import Use

>>> Schema(Use(int)).validate('123')
123

>>> Schema(Use(lambda f: open(f, 'a'))).validate('LICENSE-MIT')
<open file 'LICENSE-MIT', mode 'a' at 0x...>

```

Dropping the details, `Use` is basically:

```python
class Use(object):

    def __init__(self, callable_):
        self._callable = callable_

    def validate(self, data):
        try:
            return self._callable(data)
        except Exception as e:
            raise SchemaError('%r raised %r' % (self._callable.__name__, e))
```

Now you can write your own validation-aware classes and data types.

### Lists, similar containers

If `Schema(...)` encounteres an instance of `list`, `tuple`, `set` or
`frozenset`, it will validate contents of corresponding data container against
schemas listed inside that container:


```python
>>> Schema([1, 0]).validate([1, 1, 0, 1])
[1, 1, 0, 1]

>>> Schema(set([int, float])).validate(set([5, 7, 8, 'not int or float here']))
Traceback (most recent call last):
...
SchemaError: Or(<type 'int'>, <type 'float'>) did not validate 'not int or float here'
'not int or float here' should be instance of <type 'float'>

```

### Dictionaries

If `Schema(...)` encounters an instance of `dict`, it will validate data
key-value pairs:

```python
>>> Schema({'name': str,
...         'age': lambda n: 18 < 99}).validate({'name': 'Sue', 'age': 28}) \
... == {'name': 'Sue', 'age': 28}
True

```

You can specify keys as schemas too:

```python
>>> schema = Schema({str: int,  # string keys should have integer values
...                  int: None})  # int keys should be always None

>>> data = schema.validate({'key1': 1, 'key2': 2,
...                         10: None, 20: None})

>>> schema.validate({'key1': 1,
...                   10: 'not None here'})
Traceback (most recent call last):
...
SchemaError: key 10 is required
None does not match 'not None here'

```

You can mark a key as optional as follows:

```python
>>> from schema import Optional
>>> Schema({'name': str,
...         Optional('occupation'): str}).validate({'name': 'Sam'})
{'name': 'Sam'}

```

**schema** has classes `And` and `Or` that help to validate several schemas
for the same data:

```python
>>> from schema import And, Or

>>> Schema({'age': And(int, lambda n: 0 < n < 99)}).validate({'age': 7})
{'age': 7}

>>> Schema({'password': And(str, lambda s: len(s) > 6)}).validate({'password': 'hai'})
Traceback (most recent call last):
...
SchemaError: key 'password' is required
<lambda>('hai') should evalutate to True

>>> Schema(And(Or(int, float), lambda x: x > 0)).validate(3.1415)
3.1415

```

User-friendly error reporting
-------------------------------------------------------------------------------

You can pass a keyword argument `error` to any of validatable classes
(such as `Schema`, `And`, `Or`, `Use`) to report this error instead of
a built-in one.

```python
>>> Schema(Use(int, error='Invalid year')).validate('XVII')
Traceback (most recent call last):
...
SchemaError: Invalid year

```

You can see all errors that occured by accessing `exc.autos`
for auto-generated error messages, and `exc.errors` for errors
which had `error` text passed to them.

You can exit with `sys.exit(exc.code)` if you want to show the messages
to the user without traceback. `error` messages are given precedence in that
case.

A JSON API example
-------------------------------------------------------------------------------

Here is a quick example: validation of
[*create a gist*](http://developer.github.com/v3/gists/)
request from github API.

```python
>>> gist = '''{"description": "the description for this gist",
...            "public": true,
...            "files": {
...                "file1.txt": {"content": "String file contents"},
...                "other.txt": {"content": "Another file contents"}}}'''

>>> from schema import Schema, And, Use, Optional

>>> import json

>>> gist_schema = Schema(And(Use(json.loads),  # first convert from JSON
...                          # use basestring since json returns unicode
...                          {Optional('description'): basestring,
...                           'public': bool,
...                           'files': {basestring: {'content': basestring}}}))

>>> gist = gist_schema.validate(gist)

# gist:
{u'description': u'the description for this gist',
 u'files': {u'file1.txt': {u'content': u'String file contents'},
            u'other.txt': {u'content': u'Another file contents'}},
 u'public': True}

```

Using **schema** with [**docopt**](http://github.com/docopt/docopt)
-------------------------------------------------------------------------------

Assume you are using **docopt** with the following usage-pattern:

    Usage: my_program.py [--count=N] <path> <files>...

and you would like to validate that `<files>` are readable, and that `<path>`
exists, and that `--count` is either integer from 0 to 5, or `None`.

Assuming **docopt** returns the following dict:

```python
>>> args = {'<files>': ['LICENSE-MIT', 'setup.py'],
...         '<path>': '../',
...         '--count': '3'}

```

this is how you validate it using `schema`:

```python
>>> from schema import Schema, And, Or, Use
>>> import os

>>> s = Schema({'<files>': [Use(open)],
...             '<path>': os.path.exists,
...             '--count': Or(None, And(Use(int), lambda n: 0 < n < 5))})

>>> args = s.validate(args)

>>> args['<files>']
[<open file 'LICENSE-MIT', mode 'r' at 0x...>, <open file 'setup.py', mode 'r' at 0x...>]

>>> args['<path>']
'../'

>>> args['--count']
3

```

As you can see, **schema** validated data successfully, opened files and
converted `'3'` to `int`.

Credits
-------------------------------------------------------------------------------

This library was largely inspired by Alec Thomas'
[voluptuous](https://github.com/alecthomas/voluptuous) library, however,
**schema** tries to make it easier to use Python built-in capabilities
through lambdas, at the same time allowing to make validation-aware
classes and data types with `validate` method.
