# Parameter Nodes

## Overview

Parameter nodes are often used in scripted modules to store data such that it can be easily saved to the MRML scene. They are simply MRML nodes that exist in a scene and store data. One of their most common uses is to save GUI state in module widgets.

The parameter node concept is implemented in C++ in the `vtkMRMLScriptedModuleNode` which has `GetParameter` and `SetParameter` methods for saving arbitrary string data to the scene. While the base `vtkMRMLScriptedModuleNode` is
great for scene saving, treating all data as strings is not ideal, so a parameter node wrapper was implemented.

## Parameter Node Wrapper Classes

The parameter node wrapper allows wrapping around a `vtkMRMLScriptedModuleNode` parameter node with typed member access. A simple example is as follows.

```py
import slicer
from slicer.parameterNodeWrapper import *
from MRMLCorePython import vtkMRMLModelNode

@parameterNodeWrapper
class CustomParameterNode:
  numIterations: int
  inputs: list[vtkMRMLModelNode]
  output: vtkMRMLModelNode
```

This will create a new class called `CustomParameterNode` that has 3 members properties, an `int` named `numIterations`,
a `list` of `vtkMRMLModelNode`s named `inputs`, and a `vtkMRMLModelNode` named `output`.

The `@parameterNodeWrapper` decorator will generate a constructor for this class that takes one argument, a `vtkMRMLScriptedModuleNode` parameter node.

Usage would be as follows:

```py
parameterNode = slicer.mrmlScene.AddNewNodeByClass('vtkMRMLScriptedModuleNode')
param = CustomParameterNode(parameterNode)

# can set the property directly from a variable of appropriate type
param.numIterations = 500
param.inputs = [slicer.mrmlScene.AddNewNodeByClass('vtkMRMLModelNode') for _ in range(5)]
param.output = slicer.mrmlScene.AddNewNodeByClass('vtkMRMLModelNode')

# pythonic list usage
for inputModel in param.inputs:
  mesh = inputModel.GetMesh()
  # ...

for iteration in range(param.numIterations):
  # run iteration

param.output.SetAndObserveMesh(...)
```

### Syntax and usage

The `@parameterNodeWrapper` decorator keys off of the type hints given in the class definition, similar to python's [dataclasses.dataclass](https://docs.python.org/3/library/dataclasses.html#dataclasses.dataclass) class.

The classes that are recognized by default are

- `int`
- `float`
- `str`
- `bool`
- `vtkMRMLNode` (including subclasses)
- `list` (hinted as `list[int]`, `list[str]`, etc)
- `tuple` (hinted as `tuple[int, bool]`, `tuple[str, vtkMRMLNode, float]`, etc)
- `pathlib.Path`
- `pathlib.PosixPath`
- `pathlib.WindowsPath`
- `pathlib.PurePath`
- `pathlib.PurePosixPath`
- `pathlib.PureWindowsPath`

Only lists of these types are recognized by default. For using lists of other types see [Custom Parameter Types](#custom-parameter-types). Nested lists are available via `list[list[int]]`.

#### MRML nodes

MRML nodes from non-core modules are supported, but to define a `parameterNodeWrapper` in the global space of a module (i.e. not inside a function) you can't use the classes from `slicer` namespace. This is because they are not added to the slicer namespace until after the class is created. Here is an example on how to use `vtkMRMLMarkupsFiducialNode`.

```py
from slicer.parameterNodeWrapper import *

# Import from actual package instead of importing from "slicer"
from vtkSlicerMarkupsModuleMRMLPython import vtkMRMLMarkupsFiducialNode

@parameterNodeWrapper
class CustomParameterNode:
  markup: vtkMRMLMarkupsFiducialNode
  markups: list[vtkMRMLMarkupsFiducialNode]
```

It is possible to use core MRML classes like `vtkMRMLModelNode` from the `slicer` namespace, but in these examples we've elected to get it from `MRMLCorePython` for consistency.

#### Default values

More information can be given to the `@parameterNodeWrapper` decorator by using Annotated types.
`Annotated` is from the Python `typing` module and has the syntax
`Annotated[actualtype, annotation1, annotation2, ...]`. The `typing` module doesn't give any
annotations, so a number of type annotations were added to `slicer.parameterNodeWrapper`.
For instance, default values can be given via the `Default` annotation.

```py
from typing import Annotated
from slicer.parameterNodeWrapper import parameterNodeWrapper, Default

@parameterNodeWrapper
class CustomParameterNode:
  numIterations: Annotated[int, Default(500)]
```

This will make the default value of the `numIterations` parameter 500. If the `numIterations` parameter already has a value in the `vtkMRMLScriptedModuleNode` passed to the constructor, then that value is used instead of the default.

If a default is not set explicitly, the following values will be used:

- `int`: `0`
- `float`: `0.0`
- `str`: `""`
- `bool`: `False`
- `vtkMRMLNode` (including subclasses): `None`
- `list` (hinted as `list[int]`, `list[str]`, etc): `[]` (empty list)
- `tuple` (hinted as `tuple[int, bool]`, etc): A tuple consisting of the defaults of all the elements (e.g. `tuple[int, bool]` -> `(0, False)`)
- `pathlib.Path`: `pathlib.Path()` (which is the current directory)
- `pathlib.PosixPath`: `pathlib.PosixPath()` (which is the current directory)
- `pathlib.WindowsPath`: `pathlib.WindowsPath()` (which is the current directory)
- `pathlib.PurePath`: `pathlib.PurePath()` (which is the current directory)
- `pathlib.PurePosixPath`: `pathlib.PurePosixPath()` (which is the current directory)
- `pathlib.PureWindowsPath`: `pathlib.PureWindowsPath()` (which is the current directory)

Note: For specifying the default of a tuple, use `Annotated[tuple[int, bool], Default((4, True))]`, not `tuple[Annotated[int, Default(4)], Annotated[bool, Default(True)]]`.

#### Validators

It can be useful to restrict the set of that values passed to a parameter node. These can be done with `Validator` annotations.

```py
from typing import Annotated
from slicer.parameterNodeWrapper import parameterNodeWrapper, Minimum, Default

@parameterNodeWrapper
class CustomParameterNode:
  numIterations: Annotated[int, Minimum(0), Default(500)]

  # To have a list where the values in the list need to be validated
  chosenFeatures: list[Annotated[str, Choice(["feat1", "feat2", "feat3"])]]
```

This will cause a `ValueError` to be raised if someone tried setting `numIterations` to a negative value.

Multiple validators can be placed in the `Annotated` block and they will be run in the order they were placed.

The list of built-in validators is as follows:

- `NotNone()`
- `IsInstance(classtype)`
- `Minimum(value)` (inclusive)
- `Maximum(value)` (inclusive)
- `WithinRange(minimum, maximum)` (inclusive)
- `Choice(listOfValidChoices)` (`listOfValidChoices` can be any iterable)
- `Exclude(listOfInvalidChoices)` (`listOfInvalidChoices` can be any iterable)

The built-in types have the following validators applied to them by default:

- if "type" is `int`, `str`, `bool`, or any of the `pathlib` types:
    - `NotNone()`, `IsInstance(type)`
- `float`:
    - `NotNone()`, `IsInstance((float, int))`
    - This allows implicit conversion from `int`.
- if "type" is `vtkMRMLModelNode` or any subclass of it:
    - `IsInstance(type)`

### Custom Validators

Custom validators can easily be created and used with the `parameterNodeWrapper`.

```py
import re
from typing import Annotated
from slicer.parameterNodeWrapper import parameterNodeWrapper, Validator

# Custom validators must derive from the Validator class.
class MatchesRegex(Validator):
  def __init__(self, regex):
    self.regex = regex
  # Custom validators must implement a validate function that raises an Exception
  # if the given value is invalid.
  def validate(self, value):
    if re.match(self.regex, value) is None:
      raise ValueError("Did not match regex")

@parameterNodeWrapper
class CustomParameterNode:
  value: Annotated[str, MatchesRegex("[abc]+"), Default("abcba")]


param = CustomParameterNode(slicer.mrmlScene.AddNewNodeByClass('vtkMRMLScriptedModuleNode'))

param.value = "abcabc" # ok
param.value = "d" # ValueError raised
```

### Custom Parameter Types

#### parameterPack

It is often useful to group related information together in structures with useful names. Another decorator, `@parameterPack` was added to make this easier. This will make the class behave in a similar manner to Python's `@dataclasses.dataclass`.

```py
from slicer.parameterNodeWrapper import *

@parameterPack
class Point:
    x: float
    y: float

@parameterPack
class BoundingBox:
    # can nest parameterPacks
    topLeft: Point
    bottomRight: Point

@parameterNodeWrapper
class ParameterNodeType:
    # can add them to a @parameterNodeWrapper like any other type
    box: Annotated[BoundingBox,
        Default(BoundingBox(Point(0, 1), Point(1, 0)))]

parameterNode = slicer.mrmlScene.AddNewNodeByClass('vtkMRMLScriptedModuleNode')
param = ParameterNodeType(parameterNode)

# can set wholesale
param.box.topLeft = Point(-4, 5)

# or can set piecewise
param.box.bottomRight.x = 4
param.box.bottomRight.y = -5
```

The created `parameterPack` will have the following attributes:

```py
>>> from typing import Annotated
>>> from slicer.parameterNodeWrapper import *
>>> 
>>> @parameterPack
>>> class ParameterPack:
>>>     # if the type is Annotated, it will treat the annotations the same as @parameterNodeWrapper
>>>     x: Annotated[float, WithinRange(0, 10)]
>>>     option: Annotated[str, Choice(["a","b"]), Default("b")]
>>> 
>>> # with no arguments the constructor will use the given (or implied) defaults.
>>> p1 = ParameterPack()  # == ParameterPack(x=0.0, option="b")
>>> 
>>> # positional arguments are accepted in the order the members are declared in
>>> p2 = ParameterPack(3.0, "a")
>>> 
>>> # keyword arguments are accepted with the keyword being the member names
>>> p3 = ParameterPack(option="a", x=3.0)
>>> 
>>> # unspecified arguments use their default
>>> p4 = ParameterPack(4.5)  # == ParameterPack(x=4.5, option="b")
>>> p5 = ParameterPack(option="a")  # == ParameterPack(x=0.0, option="a")
>>> 
>>> # validators are run on construction
>>> p6 = ParameterPack(-1, "a")
ValueError: Value must be within range [0, 10], is -1
>>> 
>>> # validators are run on set attribute
>>> p4.option = "c"
ParameterPack(x=4.5, option=b)
>>> 
>>> # the classes automatically have __eq__ added to them
>>> p1 == p2
False
>>> p2 == p3
True
>>> 
>>> # the classes are also given a __repr__ and a __str__ that describes their attributes
>>> print(p4)
ParameterPack(x=4.5, option="b")
```

If any of the autogenerated dunder methods (`__init__`, `__eq__`, `__str__`, `__repr__`) are overridden in the parameterPack, they will not be autogenerated.

```py
@parameterPack
class ParameterPack:
    i: int
    j: int
    k: str

    # custom constructor
    def __init__(self, k):
        self.i = 1
        self.j = 4
        self.k = k

    # default __eq__, __str__, and __repr__ are generated
```

#### Fully Custom Parameter Types

Fully custom parameters that don't use the `parameterPack` can be supported with a little bit of work.

```py
import dataclasses
from typing import Annotated
import slicer
from slicer.parameterNodeWrapper import parameterNodeWrapper, Serializer, ValidatedSerializer

@dataclasses.dataclass
class CustomClass:
  x: int
  y: int
  z: int

# The Serializer class is used to read and write the values to the underlying 
# vtkMRMLScriptedModuleNode. There are built-in serializers for each of the support built-in types.
# Adding a new serializer involves deriving from Serializer and implementing the following methods.
# The @parameterNodeSerializer decorator registers the serializer so it can be found by a
# parameterNodeWrapper.
@parameterNodeSerializer
class CustomClassSerializer(Serializer):
  @staticmethod
  def canSerialize(type_) -> bool:
    """
    Whether the serializer can serialize the given type if it is properly instantiated.
    """
    return type_ == CustomClass

  @staticmethod
  def create(type_):
    """
    Creates a new serializer object based on the given type. If this class does not support the given type,
    None is returned.

    It is common for the returned type to actually be a ValidatedSerializer wrapping this serializer that implements
    any default validators (NotNone and IsInstance are common).
    """
    if CustomClassSerializer.canSerialize(type_):
      # in our example, lets say that we don't allow None. We will use NotNone() to enforce this
      return ValidatedSerializer(CustomClassSerializer(), [NotNone(), IsInstance(CustomClass)])
    return None

  def default(self):
    """
    The default value to use if another default is not specified.
    """
    return CustomClass(0, 0, 0)

  def isIn(self, parameterNode: slicer.vtkMRMLScriptedModuleNode, name: str) -> bool:
    """
    Whether the parameterNode contains a parameter of the given name.
    Note that most implementations can just use parameterNode.HasParameter(name).
    """
    return parameterNode.HasParameter(name)
  def write(self, parameterNode: slicer.vtkMRMLScriptedModuleNode, name: str, value) -> None:
    """
    Writes the value to the parameterNode under the given name.
    Note: It is acceptable to mangle the name as long the same name can be used for reading.
    For example the built-in ListSerializer does this.
    """
    parameterNode.SetParameter(name, f"{value.x},{value.y},{value.z}")
  def read(self, parameterNode: slicer.vtkMRMLScriptedModuleNode, name: str):
    """
    Reads and returns the value with the given name from the parameterNode.
    """
    val = parameterNode.GetParameter(name)
    vals = val.split(',')
    return CustomClass(int(vals[0]), int(vals[1]), int(vals[2]))
  def remove(self, parameterNode: slicer.vtkMRMLScriptedModuleNode, name: str) -> None:
    """
    Removes the value of the given name from the parameterNode.
    """
    parameterNode.UnsetParameter(name)

@parameterNodeWrapper
class CustomClassParameterNode(object):
  # can now use CustomClass like any other type for building parameterNodeWrappers
  custom: Annotated[CustomClass, Default(CustomClass(1,2,3))]
  listOfCustom: list[CustomClass]

```

### Caching

The `vtkMRMLScriptedModuleNode` stores its values as strings, but converting from string to another
type can be slow. Caching is used to reduce repeated costly reads under the hood. If a value hasn't been written through the _parameter node wrapper_ since the last read, then it will use the cached value. Note this means if the underlying `vtkMRMLScriptedModuleNode` parameter node changes outside of the wrapper, the cache will not be updated and the cached value will be wrong.

This is the chosen behavior for the following reasons:

- Mixed usage of the parameter node wrapper and the `vtkMRMLScriptedModuleNode` parameter node for the same parameter node is not expected.
- The `vtkMRMLScriptedModuleNode` parameter node does not offer per parameter VTK event callbacks. Therefore, if a callback was setup off the `ModifiedEvent`, _all_ parameters would be re-read for _every_ write to _any_ parameter in the node.
    - The `vtkMRMLScriptedModuleNode` parameter node may be updated in the future to give a `ParameterModifiedEvent` that gives the parameter that was modified. If this happens, the caching behavior may be revisited.


Because Python objects are returned by reference, when a cached value is returned and then modified, the modification needs to be written back to the parameter node. Otherwise, the cached value and the parameter node will get out of sync.

This write-on-change behavior has been implemented for the ListSerializer. The ListSerializer does not actually return a `list`, it returns an `ObservedList` that updates the parameter node whenever it is modified. `ObservedList` implements most `list` functions. This allows the following to work seamlessly:

```py
@parameterNodeWrapper
class ParameterNodeType(object):
  values: list[int]

param = ParameterNodeType(slicer.mrmlScene.AddNewNodeByClass('vtkMRMLScriptedModuleNode'))
values = param.values
values.append(4)
values += [7, 10]

param.values == values # True
param.values == [4, 7, 10] # True
```

The following methods are available for `ObservedList`:
- `__repr__`
- `__str__`
- `__eq__` (this will compare the contents of the list for equality)
- `__len__`
- `__getitem__`
- `__delitem__`
- `__setitem__`
- `__iadd__`
- `__imul__`
- `append`
- `extend`
- `insert`
- `remove`
- `pop`
- `clear`
- `sort`
- `reverse`

Similarly for a parameter of `list[list[type]]`, an `ObservedList[ObservedList[type]]` is returned.
When calling `param.values.append`, `param.values[index] = object` or `+=` in these cases, a normal `list` can be passed in and it will be converted to an `ObservedList`.

#### Caching for custom classes

Caching is disabled for classes that use custom serializers by default, as we do not assume that they have implemented a write-on-change functionality for their cached values. For this reason, if the `ListSerializer` uses a custom serializer for its elements, it will also disable caching by default.

If a custom serializer does implement write-on-change functionality, it can take advantage of the built-in caching mechanism quite easily (including for lists of the custom class).

```py
class CustomClassSerializer(Serializer):
  @staticmethod
  def canSerialize(type_) -> bool:
    # implementation
  @staticmethod
  def create(type_):
    # implementation
  def default(self):
    # implementation
  def isIn(self, parameterNode: slicer.vtkMRMLScriptedModuleNode, name: str) -> bool:
    # implementation
  def write(self, parameterNode: slicer.vtkMRMLScriptedModuleNode, name: str, value) -> None:
    # implementation
  def read(self, parameterNode: slicer.vtkMRMLScriptedModuleNode, name: str):
    # implementation that supports caching
  def remove(self, parameterNode: slicer.vtkMRMLScriptedModuleNode, name: str) -> None:
    # implementation

  # Add the following method override to enable caching in cases of "value: CustomClass" and "value: list[CustomClass]" (and list[list[CustomClass]] and so on).
  def supportsCaching(self):
    return True
```

## Troubleshooting

### 'slicer' has no attribute '\<MRML node name\>'

You can't use non-core MRML nodes out of the slicer namespace. See [MRML nodes](#mrml-nodes) for more info.