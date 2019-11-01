<a href="https://explosion.ai"><img src="https://explosion.ai/assets/img/logo.svg" width="125" height="125" align="right" /></a>

# catalogue: Super lightweight function registries for your library

`catalogue` is a tiny, zero-dependencies library that makes it easy to **add
function (or object) registries** to your code. Function registries are helpful
when you have objects that need to be both easily serializable and fully
customizable. Instead of passing a function into your object, you pass in an
identifier name, which the object can use to lookup the function from the
registry. This makes the object easy to serialize, because the name is a simple
string. If you instead saved the function, you'd have to use Pickle for
serialization, which has many drawbacks.

[![Azure Pipelines](https://img.shields.io/azure-devops/build/explosion-ai/public/14/master.svg?logo=azure-pipelines&style=flat-square&label=build)](https://dev.azure.com/explosion-ai/public/_build?definitionId=14)
[![Current Release Version](https://img.shields.io/github/v/release/explosion/catalogue.svg?style=flat-square&include_prereleases&logo=github)](https://github.com/explosion/catalogue/releases)
[![pypi Version](https://img.shields.io/pypi/v/catalogue.svg?style=flat-square&logo=pypi&logoColor=white)](https://pypi.org/project/catalogue/)
[![Code style: black](https://img.shields.io/badge/code%20style-black-000000.svg?style=flat-square)](https://github.com/ambv/black)

## ⏳ Installation

```python
pip install catalogue
```

`catalogue` is compatible with Python 2.7 and 3.5+.

## 👩‍💻 Usage

Let's imagine you're developing a Python package that needs to load data
somewhere. You've already implemented some loader functions for the most common
data types, but you want to allow the user to easily add their own. Using
`catalogue.create` you can create a new registry under the namespace
`your_package` &rarr; `loaders`.

```python
# YOUR PACKAGE
import catalogue

loaders = catalogue.create("your_package", "loaders")
```

This gives you a `loaders.register` decorator that your users can import and
decorate their custom loader functions with.

```python
# USER CODE
from your_package import loaders

@loaders.register("custom_loader")
def custom_loader(data):
    # Load something here...
    return data
```

The decorated function will be registered automatically and in your package,
you'll be able to access all loaders by calling `loaders.get_all`.

```python
# YOUR PACKAGE
def load_data(data, loader_id):
    print("All loaders:", loaders.get_all()) # {"custom_loader": <custom_loader>}
    loader = loaders.get(loader)
    return loader(data)
```

The user can now refer to their custom loader using only its string name
(`"custom_loader"`) and your application will know what to do and will use their
custom function.

```python
# USER CODE
from your_package import load_data

load_data(data, loader_id="custom_loader")
```

## ❓ FAQ

#### But can't the user just pass in the `custom_loader` function directly?

Sure, that's the more classic callback approach. Instead of a string ID,
`load_data` could also take a function, in which case you wouldn't need a
package like this. `catalogue` helps you when you need to produce a serializable
record of which functions were passed in. For instance, you might want to write
a log message, or save a config to load back your object later. With
`catalogue`, your functions can be parameterized by strings, so logging and
serialization remains easy – while still giving you full extensibility.

#### How do I make sure all of the registration decorators have run?

Decorators normally run when modules are imported. Relying on this side-effect
can sometimes lead to confusion, especially if there's no other reason the
module would be imported. One solution is to use
[entry points](https://packaging.python.org/tutorials/packaging-projects/#entry-points).

For instance, in [spaCy](https://spacy.io) we're starting to use function
registries to make the pipeline components much more customizable. Let's say one
user, Jo, develops a better tagging model using new machine learning research.
End-users of Jo's package should be able to write
`spacy.load("jo_tagging_model")`. They shouldn't need to remember to write
`import jos_tagged_model` first, just to run the function registries as a
side-effect. With entry points, the registration happens at install time – so
you don't need to rely on the import side-effects.

## 🎛 API

### <kbd>function</kbd> `catalogue.create`

Create a new registry for a given namespace. Returns a setter function that can
be used as a decorator or called with a name and `func` keyword argument.

| Argument     | Type       | Description                                                            |
| ------------ | ---------- | ---------------------------------------------------------------------- |
| `*namespace` | str        | The namespace, e.g. `"spacy"` or `"spacy", "architectures"`.           |
| **RETURNS**  | `Registry` | The `Registry` object with methods to register and retrieve functions. |

```python
architectures = catalogue.create("spacy", "architectures")

# Use as decorator
@architectures.register("custom_architecture")
def custom_architecture():
    pass

# Use as regular function
architectures.register("custom_architecture", func=custom_architecture)
```

### <kbd>class</kbd> `Registry`

The registry object that can be used to register and retrieve functions. It's
usually created internally when you call `catalogue.create`.

#### <kbd>method</kbd> `Registry.__init__`

Initialize a new registry.

| Argument    | Type       | Description                                                  |
| ----------- | ---------- | ------------------------------------------------------------ |
| `namespace` | Tuple[str] | The namespace, e.g. `"spacy"` or `"spacy", "architectures"`. |
| **RETURNS** | `Registry` | The newly created object.                                    |

```python
architectures = Registry(("spacy", "architectures"))
```

#### <kbd>method</kbd> `Registry.register`

Register a function in the registry's namespace. Can be used as a decorator or
called as a function with the `func` keyword argument supplying the function to
register.

| Argument    | Type     | Description                                               |
| ----------- | -------- | --------------------------------------------------------- |
| `name`      | str      | The name to register under the namespace.                 |
| `func`      | Any      | Optional function to register (if not used as decorator). |
| **RETURNS** | Callable | The decorator that takes one argument, the name.          |

```python
architectures = Registry(("spacy", "architectures"))

# Use as decorator
@architectures.register("custom_architecture")
def custom_architecture():
    pass

# Use as regular function
architectures.register("custom_architecture", func=custom_architecture)
```

#### <kbd>method</kbd> `Registry.get`

Get a function registered in the namespace.

| Argument    | Type | Description              |
| ----------- | ---- | ------------------------ |
| `name`      | str  | The name.                |
| **RETURNS** | Any  | The registered function. |

```python
custom_architecture = architectures.get("custom_architecture")
```

#### <kbd>method</kbd> `Registry.get_all`

Get all functions in the registry's namespace.

| Argument    | Type           | Description                              |
| ----------- | -------------- | ---------------------------------------- |
| **RETURNS** | Dict[str, Any] | The registered functions, keyed by name. |

```python
all_architectures = architectures.get_all()
# {"custom_architecture": <custom_architecture>}
```

### <kbd>function</kbd> `catalogue.check_exists`

Check if a namespace exists.

| Argument     | Type | Description                                                  |
| ------------ | ---- | ------------------------------------------------------------ |
| `*namespace` | str  | The namespace, e.g. `"spacy"` or `"spacy", "architectures"`. |
| **RETURNS**  | bool | Whether the namespace exists.                                |
