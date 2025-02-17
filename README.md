# palimpsest — Fast serializable C++ dictionaries

[**Installation**](https://github.com/stephane-caron/palimpsest/#installation)
| [**Documentation**](https://scaron.info/doc/palimpsest/)
| [**Example**](https://github.com/stephane-caron/palimpsest#overview)
| [**Features**](https://github.com/stephane-caron/palimpsest#features-and-non-features)
| [**Contributing**](https://github.com/stephane-caron/palimpsest#contributing)

[![Build](https://img.shields.io/github/workflow/status/stephane-caron/palimpsest/CI)](https://github.com/stephane-caron/palimpsest/actions)
![C++ version](https://img.shields.io/badge/C++-17/20-blue.svg?style=flat)
[![Release](https://img.shields.io/github/v/release/stephane-caron/palimpsest.svg?sort=semver)](https://github.com/stephane-caron/palimpsest/releases)

_palimpsest_ is a small C++ library that provides a ``Dictionary`` type meant for fast value updates and serialization. It is called [palimpsest](https://en.wiktionary.org/wiki/palimpsest#Noun) because these dictionaries are designed for frequent rewritings (values change fast) on the same support (keys change slow).

## Overview

The two main assumptions in _palimpsest_ dictionaries are that:

* **Keys** are strings.
* **Values** hold either a sub-dictionary or a type that can be unambiguously serialized.

Numbers, strings and tensors can be readily serialized, so we can straightforwardly write structures like:

```cpp
using palimpsest::Dictionary;

Dictionary world;
world("name") = "example";
world("temperature") = 28.0;

auto& bodies = world("bodies");
bodies("plane")("orientation") = Eigen::Quaterniond{0.9239, 0.3827, 0., 0.};
bodies("plane")("position") = Eigen::Vector3d{0.0, 0.0, 100.0};
bodies("truck")("orientation") = Eigen::Quaterniond::Identity();
bodies("truck")("position") = Eigen::Vector3d{42.0, 0.0, 0.0};

std::cout << world << std::endl;
```

This example outputs:

```json
{"bodies": {"truck": {"position": [42, 0.5, 0], "orientation": [1, 0, 0, 0]},
"plane": {"position": [0.1, 0, 100], "orientation": [0.9239, 0.3827, 0, 0]}},
"temperature": 28, "name": "example"}
```

We can serialize this dictionary to a binary file using the convenience ``write`` function:

```cpp
world.write("world.mpack");
```

And deserialize it likewise:

```cpp
Dictionary world_bis;
world_bis.read("world.mpack");
std::cout << world_bis << std::endl;
```

While one-time serialization to a file can be useful, dictionaries are more generally meant to be [serialized to bytes](#serialization-to-bytes) for transmission over your preferred medium (TCP connection, memory-mapped files, transcontinental telegraph line, …).

## Features and non-features

All design decisions have their pros and cons, and the ones in _palimpsest_ are rooted in the robotics applications that prompted its development. Take a look at the features and non-features below to decide if it is also a fit for _your_ use case.

### Features

* Prioritizes speed (over user-friendliness)
* References to sub-dictionaries or values help avoid key lookups
* Built for fast inter-process communication with [Python](https://www.python.org/)
* Built-in support for [Eigen](https://eigen.tuxfamily.org/)
* Serialize to and deserialize from [MessagePack](https://msgpack.org/)
* Print dictionaries to standard output as [JSON](https://www.json.org/json-en.html)
* [Extensible](#adding-custom-types) to new types (as long as they deserialize unambiguously)

### Non-features

* (Prioritizes speed) over user-friendliness
* Array values are mostly limited to Eigen tensors (matrix, quaternion, vector)
* Copy constructors are disabled
* (Extensible to new types) as long as they deserialize unambiguously
* [WIP](#contributing): key collisions are pretty much left up to the user
* [WIP](#contributing): shallow and deep copies are not implemented yet

Check out the existing [alternatives](https://github.com/stephane-caron/palimpsest#alternatives) if any of these points is a no-go for you.

## Installation

### Bazel (recommended)

You can build _palimpsest_ by running ``./tools/bazelisk build //...`` straight from the repository. To use it in your project, create a git repository rule such as:

```python
load("@bazel_tools//tools/build_defs/repo:git.bzl", "git_repository")

def palimpsest_repository():
    git_repository(
        name = "palimpsest",
        remote = "git@github.com:stephane-caron/palimpsest.git",
        commit = "...",
        shallow_since = "..."
    )
```

Complete this rull and call it from your Bazel ``WORKSPACE``. You can then use the ``@palimpsest`` dependency in your C++ rule ``deps``.

### CMake

Make sure Eigen is installed system-wise, for instance on Debian-based distributions:

```console
sudo apt install libeigen3-dev
```

Then follow the standard CMake procedure:

```console
mkdir build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release
make -j4
make install
```

Note that by default [MPack](https://github.com/ludocode/mpack) will be built and installed from the [``third_party``](https://github.com/stephane-caron/palimpsest/tree/main/third_party) folder. Set ``-DBUILD_MPACK=OFF`` if you already have MPack 1.1 or later installed on your system.

## Usage

### Serialization to bytes

Dictionaries can be serialized (``palimpsest::Dictionary::serialize``) to vectors of bytes:

```cpp
Dictionary world;
std::vector<char> buffer;
size_t size = world.serialize(buffer);
```

The function resizes the buffer automatically if needed, and returns the number of bytes of the serialized message.

### Deserialization from bytes

#### Extensions

Dictionaries can be extended (``palimpsest::Dictionary::extend``) from byte vectors:

```cpp
std::vector<char> buffer(size);
some_source.read(buffer.data(), size);

Dictionary dict;
dict.extend(buffer.data(), size);
```

A single dictionary can be extended multiple times from different sources. The catch is that key collisions are ignored [for now](#contributing), so that extending ``{"a": 12}`` with ``{"a": 42, "b": 1}`` will result in ``{"a": 12, "b": 1}`` (and a warning).

#### Updates

Dictionaries can be updated (``palimpsest::Dictionary::update``) from byte vectors rather than extended. In that case only the keys that are already in the original dictionary get new values:

```cpp
Dictionary foo;
foo("bar") = 1;
foo("foo") = 2;

Dictionary bar;
bar("bar") = 3;
std::vector<char> buffer;
size_t size = bar.serialize(buffer);

foo.update(buffer.data(), size);  // OK, now foo("bar") == 3
```

Keys in the update stream that are not already in the dictionary are ignored:

```cpp
bar("new") = 4;
size_t size = bar.serialize(buffer);
foo.update(buffer.data(), size);  // no effect
```

Updates therefore behave complementarily to extensions: updating ``{"a": 12}`` with ``{"a": 42, "b": 1}`` results in ``{"a": 42}`` rather than ``{"a": 12, "b": 1}``.

### Adding custom types

Adding a new custom type boils down to the following steps:

* Add implicit type conversions to ``Dictionary.h``
* Add a read function specialization to ``mpack/read.h``
* Add a write function specialization to ``mpack/Writer.h``
* Add a write function specialization to ``mpack/write.h``
* Add a write function specialization to ``json/write.h``

Take a look at the existing types in these files and in unit tests for inspiration.

## Q and A

Why isn't _palimpsest_ also distributed as a header-only library?

> The main blocker is that we set a custom flush function
> ``mpack_std_vector_writer_flush`` to our internal MPack writers. The [MPack
> Write API](https://ludocode.github.io/mpack/group__writer.html) requires
> a function pointer for that, and we define that function in
> [`Writer.cpp`](src/mpack/Writer.cpp). Open a PR if you have ideas to go
> around that!

## Alternatives

* [mc\_rtc::DataStore](https://github.com/jrl-umi3218/mc_rtc/blob/master/include/mc_rtc/DataStore.h) - can hold more general value types, like lambda functions, but does not serialize.
* [mjlib::telemetry](https://github.com/mjbots/mjlib/tree/master/mjlib/telemetry#readme) - if your use case is more specifically telemetry in robotics or embedded systems.
* [nlohmann::json](https://github.com/nlohmann/json) - most user-friendly library of this list, serializes to MessagePack and other binary formats, but not designed for speed.
* [Protocol Buffers](https://developers.google.com/protocol-buffers/) - good fit if you have a fixed schema (keys don't change at all) that you want to serialize to and from.
* [RapidJSON](https://github.com/Tencent/rapidjson/) - low memory footprint, can serialize to MessagePack using other [related projects](https://github.com/Tencent/rapidjson/wiki/Related-Projects), but has linear lookup complexity as it stores dictionaries [as lists of key-value pairs](https://github.com/Tencent/rapidjson/issues/102).
* [std::map](https://www.cplusplus.com/reference/map/map/) - best pick if your values all have the same type and you don't need nested dictionaries.
* [std::unordered\_map](https://www.cplusplus.com/reference/unordered_map/unordered_map/) - similar use case to ``std::map``, this variant usually performs faster on average.

## Contributing

There are many open leads to improve this project, as you already know if you landed here from a link in the README 😉 All contributions are welcome, big or small! Make sure you read the 👷 [contribution guidelines](CONTRIBUTING.md).
