# pylibyaml

pylibyaml is a simple Python module that monkey patches PyYAML to
automatically enable the fast LibYAML-based parser and emitter if they
are installed.

## Installation

To install, run:

    pip install pylibyaml

There is no explicit requirement for PyYAML or LibYAML to be installed
in advance, but this package will be useless without them. Please refer
to the PyYAML installation documentation, especially the points about
installing the LibYAML bindings.

- https://pyyaml.org/wiki/PyYAMLDocumentation
- https://pyyaml.org/wiki/LibYAML

## Usage

Run `import pylibyaml` **BEFORE** `import yaml`, and enjoy!

```python
import pylibyaml
import yaml

yaml.safe_load(stream)
yaml.load(stream, Loader=yaml.SafeLoader)

yaml.safe_dump(data)
yaml.dump(data, Dumper=yaml.SafeDumper)
```

Most existing code should run without modification. Any references to
`yaml.Loader` and `yaml.Dumper` (including `Safe`, `Unsafe`, and `Full`
flavors) will automatically point to their `yaml.cyaml.CLoader` and
`yaml.cyaml.CDumper` equivalents. The convenience methods (`safe_load`,
`safe_dump`, etc.) will all use the C classes, as well as the methods
for adding resolvers, constructors, or representers. Objects that
inherit from `YAMLObject` should work as intended.

## Details

### Background

PyYAML is the canonical YAML parser and emitter library for Python. It
is not particularly fast.

LibYAML is a C library for parsing and emitting YAML. It is very fast.

By default, the setup.py script for PyYAML checks whether LibYAML is
installed and if so, builds and installs LibYAML bindings.

For the bindings to actually be used, they need to be explicitly
selected. The PyYAML documentation suggests some variations of the
following:

> When LibYAML bindings are installed, you may use fast LibYAML-based
parser and emitter as follows:

    >>> yaml.load(stream, Loader=yaml.CLoader)
    >>> yaml.dump(data, Dumper=yaml.CDumper)

> In order to use LibYAML based parser and emitter, use the classes
> CParser and CEmitter. For instance,

    from yaml import load, dump
    try:
        from yaml import CLoader as Loader, CDumper as Dumper
    except ImportError:
        from yaml import Loader, Dumper
    # ...
    data = load(stream, Loader=Loader)
    # ...
    output = dump(data, Dumper=Dumper)    

This approach is repetitive, inconvenient, and ineffectual when dealing
with third-party libraries that also use PyYAML.

### Implementation

The approach taken by `pylibyaml` is to rebind the global names of the
Loaders and Dumpers in the `yaml` module to the LibYAML versions if they
are available, before the various functions and classes are defined.

For example, compare the following.

Without pylibyaml:

    >>> import yaml
    >>> yaml.Loader
    <class 'yaml.loader.Loader'>
    >>> yaml.Dumper
    <class 'yaml.dumper.Dumper'>
    >>> help(yaml.dump)
    Help on function dump in module yaml:

    dump(data, stream=None, Dumper=<class 'yaml.dumper.Dumper'>, **kwds)
        Serialize a Python object into a YAML stream.
        If stream is None, return the produced string instead.

Using pylibyaml (with LibYAML bindings available):

    >>> import pylibyaml
    >>> import yaml
    >>> yaml.Loader
    <class 'yaml.cyaml.CLoader'>
    >>> yaml.Dumper
    <class 'yaml.cyaml.CDumper'>
    >>> help(yaml.dump)
    Help on function dump in module yaml:

    dump(data, stream=None, Dumper=<class 'yaml.cyaml.CDumper'>, **kwds)
        Serialize a Python object into a YAML stream.
        If stream is None, return the produced string instead.

Note that the top-level names now point to the cyaml versions, and that
the default function arguments have changed.

The code samples above will still run without modification, but the second
can be simplified - the logic of determining the best loader and dumper is
not longer required.

    import pylibyaml
    from yaml import load, dump
    from yaml import Loader, Dumper
    # ...
    data = load(stream, Loader=Loader)
    # ...
    output = dump(data, Dumper=Dumper)

## Caveats

### This is a rather ugly hack.

In order need to rebind the names of the default loaders and dumpers
prior to the function and class definitions in PyYAML's `__init__.py`,
we use `inspect` to get the source, edit it, and reload it with
`importlib`. This works for now (the current version of PyYAML is
5.3.1), and as far back as 3.11, but it may not always.

### LibYAML and PyYAML are not 100% interchangeable.

From the [PyYAML docs](https://pyyaml.org/wiki/PyYAMLDocumentation):

> Note that there are some subtle (but not really significant)
> differences between pure Python and
> [LibYAML](https://pyyaml.org/wiki/LibYAML) based parsers and emitters.

