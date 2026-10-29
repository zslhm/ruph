# ruph — Rust + PHP

A single-binary web server with a built-in PHP interpreter. No PHP installation required.

ruph serves files directly, and runs PHP only where a directory controller asks for it.

- **Directory controllers.** `_index.php` runs for requests in that directory and below. It is a controller, and ruph does not serve it as a URL.
- **Cascade.** Every `_index.php` from the server root down to the request runs, parent before child. A controller that sends output, redirects, returns, or exits ends the request. The deepest file is the leaf: `return true` lets ruph deliver the file.
- **Rust delivers the file.** Static files and directory indexes are served by ruph when the cascade passes through. An existing `.php` file runs after the server-wide and vhost controllers; the local cascade does not wrap it.
- **Automatic TLS.** ruph issues and renews certificates, including wildcards, with a built-in ACME client and embedded DNS server. Port 80 stays closed.
- **One binary.** The PHP interpreter is compiled in, using tree-sitter. An external `php-cgi` is optional when a site needs full PHP.

## Quick Start

Point ruph at the folder that holds the site. That folder is the docroot. ruph serves the files in it, and runs `_index.php` only where you add one.

```bash
./ruph /var/www/mysite
```

With HTTPS:

```bash
./ruph /var/www/mysite --tls
```

With a config file:

```bash
./ruph -c ruph.ini /var/www/mysite
```

Running `./ruph` with no arguments prints help.

### Serve a page

```text
/var/www/mysite/
  index.html
```

A request for `/` serves `index.html`. Add `_index.php` later, and only in the folders where you want a rule: a redirect, an access check, a custom 404, or a log line. If that file stays silent, ruph serves the file. If it sends output, changes the status, redirects, or exits, that is the response.

### One file for the whole site

```text
/var/www/mysite/
  _index.php
  index.html
  about.html
```

### One folder

```text
/var/www/mysite/
  _index.php
  blog/
    _index.php
    post1.html
```

Every `_index.php` from the site root down to that folder runs, parent before child. A parent that sends output, redirects, returns, or exits ends the request. The deepest `_index.php` is the leaf: it can handle the request, or stay silent and let ruph serve the file.

[LATEST_CASCADE.md](LATEST_CASCADE.md) is the short map of that order. It shows which `_index.php` runs for a URL such as `https://example.com/a/b/page`, which file is the leaf, and what stops the chain.

An existing `.php` file is the exception. The site-wide `_index.php` still runs, then that file runs. `_index.php` files in between do not. A file named `_index.php` is never itself a URL.

```text
/var/www/mysite/
  contact.php
```

A request for `/contact.php` runs that file.

### More than one site

```ini
[https.example.com]
docroot = /var/www/example.com

[https.other.org]
docroot = /var/www/other.org
```

Each site has its own folder and its own `_index.php`.

### See the log

```bash
./ruph /var/www/mysite --log-console
./ruph /var/www/mysite --log-level debug --log-console
./ruph /var/www/mysite --stdoutstats 5
```

Levels are `error`, `warn`, `info` (default), `debug`, and `trace`. Without `--log-console`, a chosen log level goes to the error log file. `--stdoutstats 5` prints server stats every five seconds.

The exact request contract is in [docs/REQUESTS.md](docs/REQUESTS.md). Config keys are in [docs/RUPH_INI.md](docs/RUPH_INI.md). The built-in PHP language is in [docs/RUPH-PHP.md](docs/RUPH-PHP.md).

## How It Works

### Request Flow

```
Request arrives
    │
    ├── Select effective vhost docroot
    ├── ruph resolves URI against filesystem
    │   Sets: $_SERVER['rr_file'], rr_dir, rr_index, rr_leaf_idx, rr_mime
    │
    ├── Execute global /_index.php (master), if present
    ├── Execute vhost-root /_index.php, if present
    │
    ├── Existing .php file → execute it (local _index.php files do not run)
    ├── Any other target → run every local _index.php below the vhost root,
    │   parent before child
    │   ├── intermediate _index.php files are controllers
    │   ├── the deepest _index.php is the leaf
    │   ├── leaf handles → DONE
    │   └── leaf passes through → Rust delivers the target, or 500 if none
    │
    └── No local _index.php and no file → 404
```

### _index.php Architecture

Controllers run top to bottom. A controller that sends output, sets a redirect, returns, or exits ends the request.

1. **Global master** (`/<server-root>/_index.php`)
2. **Vhost-root controller** (`/<vhost-docroot>/_index.php`)
3. **Each intermediate** `_index.php` below the vhost root, parent before child
4. **Deepest local leaf** (`/subdir/_index.php`)

An existing `.php` file runs after the global master and the vhost-root controller. Intermediate and leaf `_index.php` files do not run for that request.

The intended model is:
- global master = server-wide policy
- vhost-root `_index.php` = site-wide policy
- intermediate `_index.php` files = policy for that directory and below
- deepest local leaf `_index.php` = local interception/routing
- otherwise Rust serves the resolved target directly

See [LATEST_CASCADE.md](LATEST_CASCADE.md) for the controller order on one URL, and [docs/REQUESTS.md](docs/REQUESTS.md) for the full request contract.

```php
<?php
// Master _index.php — one session per visitor, then delegate.
// Pass a unique id. A shared constant such as "guest" is one shared session.
$sid = $_COOKIE['RUPHSESSID'] ?? '';
if ($sid === '') {
    $sid = md5((string) microtime(true) . (string) mt_rand());
}
ruph_session_start($sid);
if (!authenticated()) { http_response_code(401); exit; }

// Let the later _index.php files handle it, or ruph serves static
```

### Pre-resolved Request Info

ruph resolves the filesystem **before** PHP runs and sets these `$_SERVER` keys:

| Key | Value |
|-----|-------|
| `rr_root` | Absolute path to the vhost document root |
| `rr_file` | Realpath of matched file, or empty |
| `rr_exists` | `"1"` if URI maps to an existing file, or empty |
| `rr_dir` | Realpath if URI maps to a directory, or empty |
| `rr_index` | First matching index file inside `rr_dir`, or empty |
| `rr_leaf_idx` | Deepest relevant local `_index.php` below the vhost root, or empty |
| `rr_mime` | MIME type ruph would use for `rr_file` |

Plus all standard `$_SERVER` keys: `REQUEST_URI`, `REQUEST_METHOD`, `QUERY_STRING`, `HTTP_HOST`, `DOCUMENT_ROOT`, `SCRIPT_FILENAME`, `PHP_SELF`, `PATH_INFO`, and all `HTTP_*` headers.

For full details on request flow, controller behavior, and `rr_*` variables, see [docs/REQUESTS.md](docs/REQUESTS.md).

## Ruph PHP Reference

ruph's built-in interpreter runs PHP natively in Rust — no external PHP binary needed. It covers the PHP features used in web templating and request handling.

### Language Features

| Feature | Support |
|---------|---------|
| `<?php ?>` and `<?= ?>` tags | Full |
| Inline HTML mixed with PHP | Full |
| Variables, assignment, string interpolation | Full |
| `if` / `else` / `elseif` | Full |
| `while`, `for`, `foreach` | Full |
| `switch` / `case` / `default` | Full |
| `break`, `continue`, `return` | Full |
| `exit` / `die` | Full |
| `include` / `require` / `_once` variants | Full |
| User-defined functions with defaults | Full |
| Arrays (indexed and associative) | Full |
| `$_GET`, `$_POST`, `$_SERVER`, `$_REQUEST` | Full |
| String concatenation (`.`), `??`, `?:` | Full |
| All comparison: `==`, `===`, `!=`, `!==`, `<`, `>`, `<=`, `>=` | Full |
| Arithmetic: `+`, `-`, `*`, `/`, `%`, `**` | Full |
| Logical: `&&`, `\|\|`, `!`, `and`, `or` | Full |
| Augmented assignment: `+=`, `-=`, `.=`, `??=` | Full |
| `++` / `--` (prefix and postfix) | Full |
| Type casting: `(int)`, `(string)`, `(bool)`, `(float)`, `(array)` | Full |
| Output buffering: `ob_start()`, `ob_get_clean()` | Full |
| `define()` / `defined()` constants | Full |

### Built-in Functions

#### String

`strlen`, `strtolower`, `strtoupper`, `trim`, `ltrim`, `rtrim`, `substr`, `str_replace`, `str_contains`, `str_starts_with`, `str_ends_with`, `strpos`, `stripos`, `strrpos`, `explode`, `implode`/`join`, `sprintf`, `nl2br`, `htmlspecialchars`, `htmlspecialchars_decode`, `htmlentities`, `urlencode`, `urldecode`, `rawurlencode`, `rawurldecode`, `ucfirst`, `lcfirst`, `str_repeat`, `str_pad`, `number_format`, `md5`

#### Array

`count`/`sizeof`, `array_keys`, `array_values`, `in_array`, `array_key_exists`, `array_merge`, `array_push`, `array_pop`, `array_slice`, `array_reverse`, `array_unique`, `array_map`, `range`, `compact`, `extract`, `sort`, `rsort`, `asort`, `arsort`, `ksort`, `krsort`

#### Type

`isset`, `empty`, `is_null`, `is_array`, `is_string`, `is_numeric`, `is_int`/`is_integer`, `is_bool`, `gettype`, `intval`, `floatval`/`doubleval`, `strval`, `boolval`

#### JSON

`json_encode`, `json_decode`

#### Filesystem

`file_exists`, `is_file`, `is_dir`, `readfile`, `file_get_contents`, `file_put_contents`, `filesize`, `dirname`, `basename`, `pathinfo`, `realpath`, `glob`

#### Date/Time

`date`, `time`, `microtime`, `strtotime`

#### Math

`abs`, `ceil`, `floor`, `round`, `max`, `min`, `rand`/`mt_rand`

#### Regex

`preg_match`, `preg_replace`, `preg_split`

#### HTTP/Output

`header`, `http_response_code`, `setcookie`, `echo`, `print_r`, `var_dump`, `error_log`

#### Misc

`phpversion`, `php_uname`, `php_sapi_name`, `function_exists`, `define`, `defined`, `constant`, `sleep`, `usleep`, `ruph_session_start`, `ruph_session_id`, `ruph_session_destroy`

#### Ruph Extensions

| Function | Description |
|----------|-------------|
| `exe($path)` | Execute a path like the web server: `exe('dir/')` runs `dir/_index.php` |
| `render($template, $data)` | Render a PHP template with data |
| `response($status, $headers, $body)` | Set response status, headers, and body in one call |
| `http_request($method, $url, $headers, $body)` | Make an HTTP request from PHP |
| `file_get_contents($url)` | Works with both local files and HTTP URLs |
| `ruph_session_start($id)` | Start a file-backed session for this Host. The id is required and should be unique per visitor. Files are mode 0600 under the system temp directory, one directory per host, so the same id on two hosts does not share state. The `RUPHSESSID` cookie is `HttpOnly` and `SameSite=Lax`. It is not marked `Secure`. |
| `ruph_session_id()` | Return the current ruph session ID |
| `ruph_session_destroy()` | Delete the current ruph session and clear `$_SESSION` |

### Pre-defined Constants

`PATHINFO_DIRNAME` (1), `PATHINFO_BASENAME` (2), `PATHINFO_EXTENSION` (4), `PATHINFO_FILENAME` (8), `PHP_EOL`, `PHP_INT_MAX`, `PHP_INT_MIN`, `DIRECTORY_SEPARATOR`, `PHP_SAPI`, `PHP_VERSION`, `TRUE`, `FALSE`, `NULL`

### What's Not Included

ruph PHP is designed for web scripting, not running legacy PHP applications. The following are **not** supported:

- Classes and objects (`class`, `new`, `->`, `::`)
- Namespaces and `use` statements
- Anonymous functions / closures
- Generators and `yield`
- Try/catch exceptions
- PHP extensions (PDO, curl, mbstring, etc.)
- Full PHP session compatibility; AST mode supports only basic `ruph_session_*` sessions

For sites that need full PHP, set `processor = cgi` in your config and ruph will use an external PHP binary as a subprocess.

## PHP Processing Modes

| Mode | How | External PHP? | Use Case |
|------|-----|---------------|----------|
| **`auto`** (default) | Built-in AST interpreter, fallback to CGI | No (optional) | Single-binary deployment |
| **`ast`** | Built-in only, no fallback | No | Guaranteed single-binary |
| **`cgi`** | External php-cgi subprocess | Yes | Full PHP compatibility |

Set via `[php] processor` in `ruph.ini`:

```ini
[php]
; auto (default) | ast | cgi
processor = auto
; Only needed for cgi mode (auto-detected if omitted)
;binary = /usr/local/bin/php-cgi
```

## CLI Options

```
Usage: ruph [OPTIONS] [DOCROOT]

Arguments:
  [DOCROOT]  Root directory to serve

Options:
      --bind-https <ADDR>         HTTPS bind address, e.g. 0.0.0.0:8082 (alias: --bind)
      --bind-http <ADDR>          Plain-HTTP bind address, e.g. 0.0.0.0:80 (alias: --http-bind)
  -c, --config <FILE>             Configuration file (INI format)
      --new-cert <EMAIL,DOMAINS>  Issue an ACME certificate
      --refresh-ssl               Refresh certificates expiring within 30 days
      --list-certs                List known certificates and exit
      --tls                       Enable TLS (uses certs from ~/.ruph/ssl)
      --php-binary <BINARY>       PHP binary for cgi mode (overrides config)
      --log-level <LEVEL>         error, warn, info, debug, trace
      --log-console               Also log to stdout
      --stdoutstats <SECONDS>     Print server stats to stdout every N seconds (0 = off)
  -h, --help                      Print help
  -V, --version                   Print version
```

## Configuration (INI)

ruph looks for `ruph.ini` in these locations (first match wins):

1. `<docroot>/ruph.ini`
2. `./ruph.ini`
3. `~/.ruph/ruph.ini`
4. `/etc/ruph/ruph.ini`

### Example ruph.ini

```ini
[server]
bind = 0.0.0.0:8082
log_level = info
tls = false
; Reject POST bodies larger than this. Plain bytes, or a K/M/G suffix. Default: 8M.
;max_body_size = 8M

[http]
docroot = /var/www/html
index_files = _index.php,index.html

[php]
; auto | ast | cgi
processor = auto
```

## TLS / ACME Certificates

Certificates are stored per-domain in `~/.ruph/ssl/<domain>/`.

```bash
# Issue a Let's Encrypt certificate
./ruph --new-cert you@example.com,example.com

# List certificates and expiry dates
./ruph --list-certs
```

Multiple domains are supported via SNI. Certificates expiring within 30 days generate a warning at startup.

## Building

```bash
cargo build --release
```

The single binary is at `target/release/ruph`.

### Cross-platform builds

```bash
# macOS (Apple Silicon)
cargo build --release --target aarch64-apple-darwin

# macOS (Intel)
cargo build --release --target x86_64-apple-darwin

# Windows
cargo build --release --target x86_64-pc-windows-msvc

# Linux (musl, fully static)
cargo build --release --target x86_64-unknown-linux-musl
```

### Running tests

```bash
cargo test
```

## License

ruph is free software, licensed under the GNU General Public License v3.0 only. See [LICENSE](LICENSE).

Copyright (C) 2026 zaunere
