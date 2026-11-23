# Getting started

## Requirements

- C++20. The library builds its arguments with designated initializers (`{.logger = ..., .on_error = ...}`), and the examples of this documentation use them everywhere.
- A compiler with `std::filesystem`: GCC, Clang and MSVC are tested.

The library is a static library, not header-only: `make_filesystem` is in `src/std_filesystem.cpp`. The target is `virtual_filesystem::virtual_filesystem`. It carries the include directory and the C++20 requirement.

## CMake

The library lives at [github.com/goloveshin/virtual_filesystem](https://github.com/goloveshin/virtual_filesystem). Fetch it with `FetchContent`:

```cmake
include(FetchContent)

FetchContent_Declare(
    virtual_filesystem
    GIT_REPOSITORY https://github.com/goloveshin/virtual_filesystem.git
    GIT_TAG main
)
FetchContent_MakeAvailable(virtual_filesystem)

target_link_libraries(my_app PRIVATE virtual_filesystem::virtual_filesystem)
```

Replace `main` with a release tag to pin a version. To use a local copy instead, replace `GIT_REPOSITORY` and `GIT_TAG` with `SOURCE_DIR <path>`.

Include one header. The headers in `detail` are not included on their own:

```cpp
#include <virtual_filesystem/filesystem.hpp>
```

The library has an option `VIRTUAL_FILESYSTEM_BUILD_TESTS`. It builds the unit tests and the example of the library. It is `ON` only when `virtual_filesystem` is the top-level project, so a project that uses the library does not build them. You can set it yourself before `FetchContent_MakeAvailable`:

```cmake
set(VIRTUAL_FILESYSTEM_BUILD_TESTS OFF)
```

## The first program

```cpp
#include <virtual_filesystem/filesystem.hpp>

#include <iostream>

namespace vfs = virtual_filesystem;
namespace fs = std::filesystem;


int main()
{
    const auto filesystem = vfs::make_filesystem({
        .logger = [](const vfs::log_t& log)
        {
            std::cout << "log: " << log.path.generic_string() << "\n";
        },
        .on_error = [](const fs::filesystem_error& error)
        {
            std::cout << "error: " << error.path1().generic_string() << "\n";
        }
    });

    const vfs::path_t path = "no_such_file.txt";

    // a missing file is not an error for status
    const auto is_there = vfs::exists(filesystem, path);
    std::cout << std::boolalpha << "exists: " << is_there << "\n";

    // file_size of a missing file is an error: on_error runs, the result is 0
    const auto size = filesystem.file_size(path);
    std::cout << "size: " << size << "\n";
}
```

It prints:

```text
log: no_such_file.txt
exists: false
log: no_such_file.txt
error: no_such_file.txt
size: 0
```

What happens here:

1. `make_filesystem` returns a `filesystem_t` whose members call `std::filesystem`. The `logger` and `on_error` functions are held by every member.
2. `exists(filesystem, path)` is a free function. It calls `filesystem.status(path)`, so the logger runs once. A missing file is `file_type::not_found` and not an error, so `on_error` does not run.
3. `filesystem.file_size(path)` throws `std::filesystem::filesystem_error` in `std::filesystem`. Here the library catches it. The logger runs first, then `on_error` receives the error. Since `on_error` returns, the call returns `0`, a value-initialized `std::uintmax_t`.

See [Errors and logging](errors.md).

## The example and the tests

The `example/` directory of the library has code under test, a filesystem in memory, and a program that checks how they work together. It builds and runs with the tests:

```text
cmake -S . -B build -G Ninja
cmake --build build
ctest --test-dir build --output-on-failure
```

| file | what it is |
| --- | --- |
| `example/config_loader.hpp` | code under test: four functions that take a `filesystem_t` |
| `example/memory_filesystem.hpp` | `memory_t`, a fake that keeps files in a `std::map` |
| `example/main.cpp` | checks: fakes, queries, errors and logging, file bytes |

The pages of the guide quote this code.
