# `_index.php` cascade

This is the order ruph runs directory controllers for one request. The example is `https://example.com/a/b/page`. The vhost docroot is the directory for `example.com`. ruph runs only the files that exist, top to bottom.

| Order | File | Role |
|------:|------|------|
| 1 | `/<server-root>/_index.php` | Global master |
| 2 | `/<server-root>/example.com/_index.php` | Vhost root |
| 3 | `/<server-root>/example.com/a/_index.php` | Intermediate controller |
| 4 | `/<server-root>/example.com/a/b/_index.php` | Leaf |

## When the chain stops

A controller (every file except the leaf) ends the request on any of these:

- output
- a `Location` header
- a non-200 status
- `exit` or `die`
- any `return`

Silence continues to the next file.

The leaf uses those same stop rules, with one exception: `return true` passes through and ruph serves the file. If the path does not exist and the leaf passes through, ruph returns 500.

## Existing `.php` files

A request for an existing `.php` file runs the global master and the vhost root, then that file. Intermediate and leaf `_index.php` files do not run.

## `rr_leaf_idx`

`rr_leaf_idx` names only the deepest `_index.php`. It does not list the controllers above it.

The full request contract is in [docs/REQUESTS.md](docs/REQUESTS.md).
