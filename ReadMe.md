casecheck - find names that will cause trouble when a tree moves to Windows.

Four kinds of problems are reported:

  1. Same directory: sibling entries whose names differ only by case, e.g.
     "Readme.txt" and "README.TXT" (Windows is case-insensitive).
  2. Merged directories: if "Foo/" and "foo/" both exist, Windows would merge
     them into one folder, so same-named entries inside them also collide
     (e.g. Foo/a.txt and foo/a.txt).
  3. Invalid names: entries Windows cannot store under that name:
       - reserved device names: CON, PRN, AUX, NUL, CONIN$, CONOUT$,
         COM1-COM9, LPT1-LPT9 and the superscript-digit variants. These are
         reserved in any letter case and with any extension, so "aux.c" and
         "nul.tar.gz" are invalid too.
       - illegal characters: < > : " \ | ? * and control characters (0-31)
       - names ending in a dot or a space (Windows silently strips these,
         so "report." would turn into "report")
       - names longer than 255 characters
  4. Paths too long: the full Windows path would reach MAX_PATH (260
     characters including the terminating NUL), or, for folders, MAX_PATH - 12
     (so an 8.3 filename can still be appended). Because the final location
     matters, the path is measured as if the scanned directory lived at
     --dest (default: C:\<name of the scanned directory>). Lengths are counted
     in UTF-16 code units, the way Windows counts them.

Logging is off by default. With --log, progress and a summary are printed to
stderr, and every error is appended, with a timestamp, to the file given by
--error-log (default: casecheck-errors.log in the current directory). Errors
are both the problems this script exists to find (case conflicts, invalid
names, paths too long) and anything that goes wrong while scanning (unreadable
directories, startup failures, crashes). The file is only created if there is at
least one error: a clean run with --log creates no file.

Exit status: 0 = no problems, 1 = problems found, 2 = error / incomplete scan.

Symlinks are never followed; they are treated as ordinary entries.
