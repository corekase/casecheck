# casecheck

`casecheck` scans a directory tree on Linux and reports names that will cause
trouble when the tree is moved to **Windows**:

- files and folders whose names differ only by case (Windows is case-insensitive),
- names Windows cannot store (reserved device names, illegal characters, trailing
  dots or spaces, over-long names),
- paths that would exceed Windows' path-length limit.

The script is read-only: it never renames, moves or deletes anything.

Output is one tab-separated record per line, so people can read it and other
programs can parse it. The same lines can optionally be written to a log file.

## Requirements

- Python 3.7 or newer
- Standard library only (no packages to install)
- Written for Linux and other POSIX systems

## Quick start

```
python3 casecheck.py /path/to/directory
```

or, since the script has a shebang line:

```
chmod +x casecheck.py
./casecheck.py /path/to/directory
```

Fix whatever it reports and run it again until it exits with status 0 and shows
no records.

## What it checks

Every problem is reported as a record with one of these tags.

| Tag | Meaning |
|-----|---------|
| `CASE` | Entries in the **same directory** whose names differ only by case, such as `Readme.txt` and `README.TXT`. A file and a folder count too (`docs` and `Docs`). |
| `MERGE` | Same-named entries **inside directories that case-collide**. If `Foo/` and `foo/` both exist, Windows merges them into one folder, so `Foo/a.txt` and `foo/a.txt` collide, and so do `Foo/A.txt` and `foo/a.txt`. This applies at any depth. |
| `NAME` | A name Windows can't use. See [Name rules](#name-rules). |
| `LENGTH` | The full Windows path would be too long. See [Path length](#path-length). |
| `ERROR` | Something went wrong while scanning: an unreadable directory, a scan target that isn't a directory, or a crash. |

### Case comparison

Names are compared the way Windows (NTFS) compares them: character by
character using a simple uppercase mapping. This is deliberately **not** Python's
`casefold()`, so `ß` and `ss` are treated as different names, because Windows
treats them as different.

Whole paths are compared, one component at a time. That is how `MERGE` conflicts
are found: `Foo/sub/z` and `foo/SUB/z` fold to the same path, so both are reported.

### Name rules

A `NAME` record lists every rule the name breaks, as comma-separated codes:

| Code | Meaning |
|------|---------|
| `reserved=NAME` | The name is a reserved Windows device name: `CON`, `PRN`, `AUX`, `NUL`, `CONIN$`, `CONOUT$`, `COM1`-`COM9`, `LPT1`-`LPT9`, or the superscript variants `COM¹ COM² COM³ LPT¹ LPT² LPT³`. The match ignores letter case and any extension, so `aux.c`, `NUL.tar.gz` and `con.txt` are all reserved. Trailing spaces before the extension are ignored (`nul .txt`). |
| `illegal=CHARS` | The name contains one or more of `< > : " \ | ? *` or a control character (codes 0-31). Control characters are written as `\xNN`. (`/` can't occur in a Linux filename, so it isn't checked.) |
| `trailing-dot` | The name ends with `.`. Windows silently strips it, so `report.` would become `report`. |
| `trailing-space` | The name ends with a space. Windows strips it too. |
| `name-length=N` | The name is `N` characters long, over the NTFS limit of 255. |

### Path length

Windows' classic limit (`MAX_PATH`) is 260 characters including the terminating
NUL character, and folders are held to 12 fewer so that an 8.3 short filename can
still be added. At the defaults this means:

- a **file** path may be at most **259** characters,
- a **folder** path may be at most **247** characters.

Because the limit applies to the *full* path on the Windows machine, the script
needs to know where the tree will end up. It measures each path as if the scanned
directory lived at `--dest`. If you don't give one, it assumes
`C:\<name of the scanned directory>`; scanning `/data/project` is measured as if
it were at `C:\project`. Lengths are counted in UTF-16 code units, the way
Windows counts them (an emoji counts as 2).

A `LENGTH` record reports `len=N,limit=N`, where `len` is the full Windows path
length and `limit` is the longest length allowed for that kind of entry.

## Usage

```
casecheck.py [-h] [--siblings-only] [--dest WINPATH] [--max-path N]
             [--log] [--error-log FILE]
             directory
```

| Argument | Description |
|----------|-------------|
| `directory` | The directory to scan, recursively. |
| `--siblings-only` | Report only `CASE` conflicts (same directory). Suppresses `MERGE` records, so conflicts that arise only when case-colliding directories merge are not reported. |
| `--dest WINPATH` | The Windows path where the scanned directory will end up, used for path-length checks, e.g. `--dest 'C:\Users\me\projects\mydir'`. Quote it so the shell doesn't eat the backslashes. Default: `C:\<name of the scanned directory>`. |
| `--max-path N` | The Windows `MAX_PATH` to check against, counting the terminating NUL. Default `260`. Files may be `N - 1` characters long and folders `N - 12 - 1`. Raise it (for example `--max-path 32767`) if long-path support is enabled on the target machine. Note that some Windows programs still enforce 260 even then. |
| `--log` | Turn on logging. Off by default. See [Logging](#logging). |
| `--error-log FILE` | The file `--log` writes to. Default `casecheck-errors.log` in the current directory. Requires `--log`. |

## Output format

Each record is one line of four fields separated by single **tab** characters:

```
TAG <TAB> TYPE <TAB> PATH <TAB> DETAIL
```

| Field | Contents |
|-------|----------|
| `TAG` | `CASE`, `MERGE`, `NAME`, `LENGTH` or `ERROR`. |
| `TYPE` | `dir`, `file` or `link` (a symbolic link). `-` when it doesn't apply (for example, a crash). |
| `PATH` | The path relative to the scanned directory. |
| `DETAIL` | `CASE`, `MERGE`: `group=N`. `NAME`: the codes above. `LENGTH`: `len=N,limit=N`. `ERROR`: the error message. |

Entries that share a `group=N` collide with each other. Group numbers run
consecutively across `CASE` and `MERGE` records, so a number identifies one
conflict. `CASE` conflicts are listed first, then `MERGE`, then `NAME`, then
`LENGTH` (worst overage first), then `ERROR`.

Lines starting with `#` are comments that parsers can skip. The last line of a
completed scan is always a summary:

```
# summary case=2 merge=1 name=2 length=1 error=1 dest=C:\project max_path=260
```

`case` and `merge` count conflict **groups**; `name`, `length` and `error` count
records. The summary also records the destination and `MAX_PATH` that the path
lengths were measured against.

### Example

Columns are aligned here for reading; in the real output the fields are separated
by single tabs.

```
$ python3 casecheck.py /data/project
CASE    dir   Docs                     group=1
CASE    file  docs                     group=1
CASE    dir   Foo                      group=2
CASE    dir   foo                      group=2
MERGE   file  Foo/a.txt                group=3
MERGE   file  foo/a.txt                group=3
NAME    file  src/aux.c                reserved=AUX
NAME    file  notes?.txt               illegal=?
LENGTH  file  build/deeply/nested/.../report.txt   len=263,limit=259
ERROR   dir   private                  Permission denied
# summary case=2 merge=1 name=2 length=1 error=1 dest=C:\project max_path=260
```

### Odd filenames

- Tabs, newlines and other control characters in names appear as `\xNN`
  (a tab is `\x09`), so a record is always exactly one line with exactly four fields.
- Bytes in a filename that aren't valid UTF-8 appear as `\udcNN` escapes.
- Trailing spaces in a path are kept. Don't strip whitespace from fields when
  parsing.

### Parsing the output

With `awk`:

```
python3 casecheck.py /data/project | awk -F'\t' '$1 == "NAME" { print $3 }'
```

With Python:

```python
import subprocess

result = subprocess.run(
    ["python3", "casecheck.py", "/data/project"],
    capture_output=True, text=True, encoding="utf-8",
)
for line in result.stdout.split("\n"):
    if not line or line.startswith("#"):
        continue
    tag, entry_type, path, detail = line.split("\t")
    print(tag, entry_type, path, detail)
```

## Logging

Logging is **off by default**. With `--log`, everything the script prints to the
console is also appended, unchanged, to a log file, so the console and the file
always match line for line (on a UTF-8 terminal they are byte-for-byte identical).
There are no timestamps or log levels in the file, which keeps it in the same
parseable format.

```
python3 casecheck.py /data/project --log
python3 casecheck.py /data/project --log --error-log /var/tmp/scan.txt
```

- **The file is only created if there is something to log.** If there is at least
  one problem or error, the file is created. A clean run creates no file and prints
  only the `# summary` line.
- **The file is appended to,** never overwritten. A completed scan ends with a
  `# summary` line, which marks the end of each run.
- **The default location is the current directory,** not the scanned directory.
- **The path is checked before the scan starts.** If the file can't be written (the
  directory doesn't exist, no permission, or the path is a directory), the script
  exits with status 2 immediately instead of discovering it afterwards.
- `--error-log` without `--log` is a usage error, so the option can't be silently
  ignored.
- Crashes and a missing scan target are logged too (see below).

## Streams

| Stream | Contents |
|--------|----------|
| stdout | All records, including `ERROR` records for unreadable directories, and the `# summary` line. |
| stderr | Fatal problems only: a scan target that isn't a directory, or a crash. Both are written as an `ERROR` record. A crash adds the Python traceback as `#` comment lines. Usage errors from bad arguments are also printed to stderr. |

When logging is on, the log file receives exactly what was printed to whichever
stream was used.

## Exit status

| Status | Meaning |
|--------|---------|
| `0` | No problems found. |
| `1` | At least one `CASE`, `MERGE`, `NAME` or `LENGTH` problem was found. |
| `2` | No problems found but the scan was incomplete (some directories couldn't be read), or the script couldn't run (bad arguments, not a directory, unwritable log file, crash). |

This makes it easy to use in scripts and CI:

```
python3 casecheck.py build/ --log --error-log casecheck.txt || echo "Fix the names above"
```

## Function reference

The script is a single file with no dependencies beyond the standard library.

### Name rules

| Function | Purpose |
|----------|---------|
| `fold(name)` | Case-folds a name the way Windows compares names (per-character simple uppercase mapping). |
| `reserved_name(name)` | Returns the reserved Windows device name that `name` maps to (`AUX`, `COM1`, ...), or `None`. Looks only at the part before the first `.` and ignores trailing spaces. |
| `invalid_codes(name)` | Returns the list of reasons Windows can't use a name: `reserved=...`, `illegal=...`, `trailing-dot`, `trailing-space`, `name-length=N`. |
| `wlen(text)` | Length of a string in UTF-16 code units, which is how Windows counts characters. |

### Scanning

| Function | Purpose |
|----------|---------|
| `scan(root, errors, dest, max_path)` | Walks the tree without following symlinks. Returns `(groups, invalid, too_long)`: entries grouped by case-folded full path, names with `invalid_codes`, and paths over the length limit. Unreadable directories are appended to `errors`. |
| `kind(path)` | Returns `link`, `dir` or `file` for an entry. |

### Output

| Function | Purpose |
|----------|---------|
| `printable(text)` | Escapes control characters as `\xNN` so a name can't break a record. |
| `record(tag, entry_type, path, detail)` | Builds one tab-separated output line. |
| `error_path(err, root)` | Returns the path of an unreadable directory relative to the scanned directory. |
| `build_records(root, same_dir, merged, invalid, too_long, errors)` | Turns the scan results into output lines, numbering the conflict groups and ordering them by tag. |
| `emit(lines, stream=None)` | Prints lines to the console (stdout by default) and, when logging is on, appends the same lines to the log file. Everything the script reports goes through it, which is what keeps the console and the log identical. |

### Logging

| Function | Purpose |
|----------|---------|
| `check_error_log_path(path)` | Returns a message if the log file can't be written, otherwise `None`. |
| `setup_logging(error_log)` | Turns logging on. The file is opened lazily, so it only appears when the first line is written. |
| `SafeFormatter` | A log formatter that keeps lines valid UTF-8 even when a filename contains undecodable bytes. |

### Command line

| Function | Purpose |
|----------|---------|
| `main()` | Parses arguments, runs the scan, groups the results into `CASE` and `MERGE` conflicts, prints the records and summary, and returns the exit status. |

### Constants

Near the top of the script:

| Constant | Value | Meaning |
|----------|-------|---------|
| `RESERVED_NAMES` | `CON`, `PRN`, ... | Reserved Windows device names. |
| `ILLEGAL_CHARS` | `< > : " \ \| ? *` and codes 0-31 | Characters Windows doesn't allow in names. |
| `MAX_COMPONENT` | `255` | Longest single name on NTFS. |
| `DEFAULT_MAX_PATH` | `260` | Windows `MAX_PATH`, counting the terminating NUL. |
| `DIR_PATH_MARGIN` | `12` | How much shorter folder paths must be, to leave room for an 8.3 filename. |
| `DEFAULT_ERROR_LOG` | `casecheck-errors.log` | Default log file name. |

## Notes and limitations

- **Only names are checked.** The script doesn't look at file contents, permissions,
  sizes or timestamps.
- **Symlinks are not followed.** They are reported as ordinary entries with type
  `link`.
- **Default Windows behaviour is assumed.** Windows can mark individual
  directories as case-sensitive (for example for use with WSL). The `CASE` and
  `MERGE` checks don't apply inside such directories.
- **Mixed groups get one label.** If `Mix/a.txt`, `Mix/A.TXT` and `mix/a.txt`
  all exist, the three are reported as a single `MERGE` group, including the
  same-folder pair `Mix/a.txt` and `Mix/A.TXT`, which is not also reported as `CASE`.
- **The folder length rule is applied conservatively.** A folder path plus its NUL
  must stay within `MAX_PATH - 12` (248 at the default). If you find a different
  boundary on your target, adjust `DIR_PATH_MARGIN`.
- **The destination is an assumption.** Path lengths are only as accurate as
  `--dest`. Give the real location for reliable `LENGTH` results.
