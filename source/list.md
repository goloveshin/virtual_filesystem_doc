# Listing a directory

`list` returns the entries of a directory in one vector. It replaces both `std::filesystem::directory_iterator` and `recursive_directory_iterator`.

```cpp
struct list_options_t
{
    bool recursive = false;
    bool follow_directory_symlink = false;
    bool skip_permission_denied = false;
};

struct list_t
{
    path_t root;
    list_options_t options = {};
};

using list_ft = std::function<entries_t (const list_t&)>;
```

## Arguments

`root` is the directory. `options` selects how to walk it:

| option | meaning |
| --- | --- |
| `recursive` | go into subdirectories. The default is the entries of `root` only |
| `follow_directory_symlink` | go into a symlink to a directory. It has an effect only with `recursive` |
| `skip_permission_denied` | skip a directory that cannot be opened, instead of an error |

The last two are `std::filesystem::directory_options` of the same names.

The call takes one struct, so a call site names what it sets:

```cpp
const auto entries = filesystem.list({
    .root = root,
    .options = {.recursive = true}
});
```

## The result

```cpp
struct entry_t
{
    path_t path;
    fs::file_status status;
    fs::file_status symlink_status;
};
using entries_t = std::vector<entry_t>;
```

An entry is a **snapshot**. `status` and `symlink_status` are captured when the directory is listed, so the code that goes through the entries does not ask the filesystem again. In `std::filesystem` the same questions to `directory_entry` can touch the disk and can fail.

`status` follows a symlink and `symlink_status` does not, the same as in the [queries](queries.md).

The order of the entries is the order of the directory. It is not sorted. Sort the result when the order matters.

## Code under test

`source_files` of the example takes the `.cpp` files under a root:

```cpp
inline std::vector<std::string> source_files(
    const virtual_filesystem::filesystem_t& filesystem,
    const virtual_filesystem::path_t& root
)
{
    namespace fs = virtual_filesystem::fs;

    const auto entries = filesystem.list({
        .root = root,
        .options = {.recursive = true}
    });

    std::vector<std::string> result;
    for (const auto& entry : entries)
    {
        if (entry.status.type() == fs::file_type::regular && entry.path.extension() == ".cpp")
        {
            result.push_back(entry.path.generic_string());
        }
    }
    std::ranges::sort(result);
    return result;
}
```

## A fake list

A test sets `list` and nothing else. The entries carry their own status, so no `status` fake is needed:

```cpp
virtual_filesystem::filesystem_t filesystem;
filesystem.list = [](const virtual_filesystem::list_t& args)
{
    const auto regular = fs::file_status{fs::file_type::regular};
    return virtual_filesystem::entries_t{
        {.path = args.root / "a.cpp", .status = regular, .symlink_status = regular},
        {.path = args.root / "b.hpp", .status = regular, .symlink_status = regular},
        {.path = args.root / "sub" / "c.cpp", .status = regular, .symlink_status = regular}
    };
};

example::source_files(filesystem, "src");  // {"src/a.cpp", "src/sub/c.cpp"}
```

The memory filesystem of the example (`memory_t`) fills `list` from its files too, and honors `recursive`. See [Replacing calls in a test](faking.md).

## Errors

If the directory cannot be opened or is not a directory, `std::filesystem` throws `filesystem_error`. With a filesystem from `make_filesystem` it goes to `on_error`, and the result is an empty vector:

```cpp
const auto filesystem = virtual_filesystem::make_filesystem({});
filesystem.list({.root = "missing_dir"}).size();  // 0
```

An error in the middle of a walk, for example a directory that cannot be opened without `skip_permission_denied`, also ends the call, and entries found before the error are not returned.
