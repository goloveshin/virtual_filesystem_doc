# Reference

The header is `<virtual_filesystem/filesystem.hpp>`. Everything is in `namespace virtual_filesystem`. The headers in `detail` are included only from it.

Every member of `filesystem_t` can throw `std::filesystem::filesystem_error`. A filesystem from `make_filesystem` passes it to `on_error` instead, see [Errors and logging](errors.md).

The suffixes: `_t` a type, `_ft` a `std::function`, `_et` an enumeration, `_ot` a `std::optional`.

## Aliases and values

| name | type |
| --- | --- |
| `fs` | `namespace fs = std::filesystem` |
| `path_t` | `std::filesystem::path` |
| `bytes_t` | `std::string`, the bytes of a file |
| `bytes_ot` | `std::optional<bytes_t>` |
| `entries_t` | `std::vector<entry_t>` |

### `entry_t`

```cpp
struct entry_t
{
    path_t path;
    fs::file_status status;          // follows a symlink
    fs::file_status symlink_status;  // does not
};
```

One directory entry, captured when it was listed.

### `list_options_t` and `list_t`

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
```

See [Listing a directory](list.md).

## `filesystem_t`

```cpp
struct filesystem_t;
```

Every member is a `std::function`. A default-constructed `filesystem_t` has every member empty, and calling an empty member throws `std::bad_function_call`.

### Examining

| member | signature | result |
| --- | --- | --- |
| `status` | `file_status (const path_t&)` | the file status, a symlink is followed. A missing file is `not_found` |
| `symlink_status` | `file_status (const path_t&)` | the status, a symlink is not followed |
| `file_size` | `uintmax_t (const path_t&)` | the size in bytes |
| `hard_link_count` | `uintmax_t (const path_t&)` | the number of hard links |
| `last_write_time` | `file_time_type (const path_t&)` | the time of the last write |
| `space` | `space_info (const path_t&)` | `capacity`, `free`, `available` |
| `equivalent` | `bool (const equivalent_t&)` | `true` when `path_1` and `path_2` name the same file |
| `read_symlink` | `path_t (const path_t&)` | the target of a symlink |

```cpp
struct equivalent_t
{
    path_t path_1;
    path_t path_2;
};
```

### Paths

| member | signature | result |
| --- | --- | --- |
| `current_path` | `path_t ()` | the current directory |
| `set_current_path` | `void (const path_t&)` | changes the current directory |
| `absolute` | `path_t (const path_t&)` | the absolute path |
| `canonical` | `path_t (const path_t&)` | the absolute path without symlinks, `.` and `..`. The file must exist |
| `weakly_canonical` | `path_t (const path_t&)` | the same, and the file may be missing |
| `temp_directory` | `path_t ()` | the directory for temporary files |

### Changing

| member | signature | result |
| --- | --- | --- |
| `create_directory` | `bool (const path_t&)` | `true` when the directory was created |
| `create_directories` | `bool (const path_t&)` | `true` when at least one directory was created |
| `remove` | `bool (const path_t&)` | `true` when the file or the empty directory was removed |
| `remove_all` | `uintmax_t (const path_t&)` | how many files and directories were removed |
| `rename` | `void (const rename_t&)` | moves `from` to `to` |
| `resize_file` | `void (const resize_file_t&)` | changes the size of a file |
| `copy` | `void (const copy_t&)` | copies `from` to `to` |
| `create_symlink` | `void (const create_symlink_t&)` | makes a link `link` to `target` |
| `create_directory_symlink` | `void (const create_symlink_t&)` | the same, for a directory target |
| `create_hard_link` | `void (const create_hard_link_t&)` | makes a hard link |
| `permissions` | `void (const permissions_t&)` | changes the permissions |
| `set_last_write_time` | `void (const set_last_write_time_t&)` | sets the time of the last write |

```cpp
struct rename_t { path_t from; path_t to; };
struct resize_file_t { path_t path; std::uintmax_t new_size = 0; };
struct copy_t { path_t from; path_t to; fs::copy_options options = fs::copy_options::none; };
struct create_symlink_t { path_t target; path_t link; };
struct create_hard_link_t { path_t target; path_t link; };
struct permissions_t
{
    path_t path;
    fs::perms permissions = fs::perms::none;
    fs::perm_options options = fs::perm_options::replace;
};
struct set_last_write_time_t { path_t path; fs::file_time_type time; };
```

`copy_options` selects recursion, overwrite, and how symlinks are copied, the same as in `std::filesystem::copy`.

### Listing

| member | signature | result |
| --- | --- | --- |
| `list` | `entries_t (const list_t&)` | the entries of a directory |

See [Listing a directory](list.md).

### File bytes

| member | signature | result |
| --- | --- | --- |
| `try_read` | `bytes_ot (const path_t&)` | the bytes, or an empty optional when the file cannot be opened. Not an error |
| `read` | `bytes_t (const path_t&)` | the bytes. An error when the file cannot be opened |
| `write` | `void (const write_t&)` | replaces the bytes of the file |
| `content_filter` | `bytes_t (const path_t&, const bytes_t&, content_filter_direction_et)` | changes the bytes on the way, see [File bytes and content_filter](bytes.md) |

```cpp
struct write_t
{
    path_t path;
    bytes_t bytes;
};

enum class content_filter_direction_et { READ, WRITE };
```

See [File bytes and content_filter](bytes.md).

## `make_filesystem`

```cpp
struct make_filesystem_t
{
    logger_ft logger;
    error_ft on_error;
    content_filter_ft content_filter;
};

filesystem_t make_filesystem(const make_filesystem_t&);
```

Fills every member from `std::filesystem`. `try_read`, `read` and `write` use `std::ifstream` and `std::ofstream` in binary mode.

| field | type | meaning |
| --- | --- | --- |
| `logger` | `std::function<void (const log_t&)>` | runs before every call. An empty one is skipped |
| `on_error` | `std::function<void (const fs::filesystem_error&)>` | receives a `filesystem_error` of a call. An empty one is skipped |
| `content_filter` | `content_filter_ft` | applied to `try_read`, `read` and `write`. An empty one leaves the bytes as they are |

When `on_error` returns, the call returns a value-initialized result. When `on_error` throws, that exception leaves the call. See [Errors and logging](errors.md).

### `log_t`

```cpp
struct log_t
{
    operation_et operation = operation_et::STATUS;
    path_t path;
    path_t other;
};
```

`path` is the file the call names. `other` is set when the call names two paths: `RENAME`, `COPY`, `CREATE_SYMLINK`, `CREATE_DIRECTORY_SYMLINK`, `CREATE_HARD_LINK` and `EQUIVALENT`.

### `operation_et`

```cpp
enum class operation_et
{
    STATUS, SYMLINK_STATUS, FILE_SIZE, HARD_LINK_COUNT, LAST_WRITE_TIME, SPACE,
    EQUIVALENT, READ_SYMLINK,
    CURRENT_PATH, SET_CURRENT_PATH, ABSOLUTE_PATH, CANONICAL, WEAKLY_CANONICAL, TEMP_DIRECTORY,
    CREATE_DIRECTORY, CREATE_DIRECTORIES, REMOVE, REMOVE_ALL, RENAME, RESIZE_FILE, COPY,
    CREATE_SYMLINK, CREATE_DIRECTORY_SYMLINK, CREATE_HARD_LINK, PERMISSIONS, SET_LAST_WRITE_TIME,
    LIST, TRY_READ, READ, WRITE
};
```

One value for each member of `filesystem_t` except `content_filter`. `ABSOLUTE_PATH` is not named `ABSOLUTE`: `wingdi.h` defines that macro.

## `apply_content_filter`

```cpp
void apply_content_filter(filesystem_t&);
```

Wraps `try_read`, `read` and `write` of the filesystem with its `content_filter`. An empty filter changes nothing. `make_filesystem` calls it before it returns; call it yourself only for a `filesystem_t` that you filled by hand, and only once.

## Queries

Free functions over the members. Every one takes `const filesystem_t&` and a path, and is `inline`. See [Queries](queries.md).

| function | result |
| --- | --- |
| `bool exists(filesystem, path)` | `status(path)` is not `none` and not `not_found` |
| `bool is_regular_file(filesystem, path)` | `status(path)` is `regular` |
| `bool is_directory(filesystem, path)` | `status(path)` is `directory` |
| `bool is_block_file(filesystem, path)` | `status(path)` is `block` |
| `bool is_character_file(filesystem, path)` | `status(path)` is `character` |
| `bool is_fifo(filesystem, path)` | `status(path)` is `fifo` |
| `bool is_socket(filesystem, path)` | `status(path)` is `socket` |
| `bool is_other(filesystem, path)` | the file exists and is not regular, not a directory and not a symlink |
| `bool is_symlink(filesystem, path)` | `symlink_status(path)` is `symlink` |
| `bool is_empty(filesystem, path)` | a directory has no entries, or a file has size 0 |
| `path_t relative(filesystem, path, base)` | `lexically_relative` of the weakly canonical paths |
| `path_t proximate(filesystem, path, base)` | `lexically_proximate` of the weakly canonical paths |

## CMake

| name | meaning |
| --- | --- |
| `virtual_filesystem::virtual_filesystem` | the static library target, with the include directory and C++20 |
| `VIRTUAL_FILESYSTEM_BUILD_TESTS` | builds the unit tests and the example. `ON` only for the top-level project |
