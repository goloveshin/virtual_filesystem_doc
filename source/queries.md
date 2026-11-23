# Queries

`exists`, `is_regular_file`, `is_directory`, `is_empty`, `relative` and the other questions that `std::filesystem` answers are **free functions** of the library. The first argument is the filesystem:

```cpp
virtual_filesystem::exists(filesystem, path);
virtual_filesystem::is_directory(filesystem, path);
virtual_filesystem::relative(filesystem, path, base);
```

They are not members of `filesystem_t`, because they are rules. Each one is a few lines over the members, and it is the same for every filesystem:

- the disk: `make_filesystem` fills the members, the rules are the same;
- a fake: the test sets `status`, and `exists` answers from it.

A fake cannot disagree with a rule, because the fake has no way to replace it. A test that says "this path is a directory" sets one `status`, and `is_directory`, `exists`, `is_regular_file` and `is_other` all follow.

## What each query reads

| query | reads | true when |
| --- | --- | --- |
| `exists` | `status` | the type is not `none` and not `not_found` |
| `is_regular_file` | `status` | the type is `regular` |
| `is_directory` | `status` | the type is `directory` |
| `is_block_file` | `status` | the type is `block` |
| `is_character_file` | `status` | the type is `character` |
| `is_fifo` | `status` | the type is `fifo` |
| `is_socket` | `status` | the type is `socket` |
| `is_other` | `status` | the file exists and is not regular, not a directory and not a symlink |
| `is_symlink` | `symlink_status` | the type is `symlink` |
| `is_empty` | `status`, then `list` or `file_size` | a directory has no entries, or a file has size 0 |
| `relative` | `weakly_canonical` | see below |
| `proximate` | `weakly_canonical` | see below |

All of them are `inline` and take `const filesystem_t&` and `const path_t&`. A missing member that the query needs throws `std::bad_function_call`.

## status and symlink_status

`status` follows a symlink and tells what the link points to. `symlink_status` does not follow it and tells what the link is. All the type queries except `is_symlink` use `status`:

```cpp
virtual_filesystem::filesystem_t filesystem;
filesystem.status = [](const virtual_filesystem::path_t&)
{
    return fs::file_status{fs::file_type::regular};
};
filesystem.symlink_status = [](const virtual_filesystem::path_t&)
{
    return fs::file_status{fs::file_type::symlink};
};

virtual_filesystem::is_regular_file(filesystem, "l");  // true: a link to a regular file
virtual_filesystem::is_symlink(filesystem, "l");       // true: and it is a link
```

This is one file and two answers, which is what a real symlink looks like. To build it on a disk you would need a user that is allowed to create symlinks.

## is_empty

`is_empty` asks `status` first.

- A **directory** is empty when `list({.root = path})` has no entries. The list is not recursive, and it is the call that a logger and a fake see.
- Anything else is empty when `file_size(path)` is 0.

```cpp
virtual_filesystem::filesystem_t filesystem;
filesystem.status = [](const virtual_filesystem::path_t&)
{
    return fs::file_status{fs::file_type::directory};
};
filesystem.list = [](const virtual_filesystem::list_t&)
{
    return virtual_filesystem::entries_t{};
};

virtual_filesystem::is_empty(filesystem, "d");  // true
```

!!! warning
    A missing path is not a directory, so `is_empty` calls `file_size`. On a filesystem from `make_filesystem` that fails with `filesystem_error`; if `on_error` returns, `file_size` returns 0 and `is_empty` answers `true`. Check `exists` first when "missing" must be different from "empty".

## relative and proximate

Both make the two paths weakly canonical with the filesystem, and then cut the second from the first by their names only:

```cpp
// path relative to base
path_t relative(const filesystem_t&, const path_t& path, const path_t& base);

// path proximate to base: the path itself, when there is no relative path
path_t proximate(const filesystem_t&, const path_t& path, const path_t& base);
```

`relative` is `lexically_relative` of the canonical paths, `proximate` is `lexically_proximate`.

```cpp
virtual_filesystem::filesystem_t filesystem;
filesystem.weakly_canonical = [](const virtual_filesystem::path_t& path)
{
    return path;
};

relative(filesystem, "/a/b/c", "/a");    // "b/c"
relative(filesystem, "/x/y", "/a");      // "../x/y"
relative(filesystem, "x/y", "/a");       // "": a relative path and an absolute base have no relative path
proximate(filesystem, "x/y", "/a");      // "x/y": the path itself
```

The fake `weakly_canonical` returns the path as it is, so the test does not depend on the disk. On the disk the paths are resolved first: symlinks are followed and `.` and `..` are removed, so two names of one place give one answer.

## Errors

The queries do not catch anything. A member that throws `filesystem_error` makes the query throw it. With a filesystem from `make_filesystem` the member does not throw: it logs, calls `on_error`, and returns an empty result. See [Errors and logging](errors.md).
