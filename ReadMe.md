casecheck - find names that will cause trouble when a tree moves to Windows.

Problems are reported with one of these tags:

  CASE    entries in the same directory whose names differ only by case, e.g.
          "Readme.txt" and "README.TXT" (Windows is case-insensitive).
          
  MERGE   same-named entries inside directories that case-collide: if "Foo/"
          and "foo/" both exist, Windows merges them, so Foo/a.txt and
          foo/a.txt collide too.
          
  NAME    names Windows cannot store: reserved device names (CON, PRN, AUX,
          NUL, CONIN$, CONOUT$, COM1-COM9, LPT1-LPT9 and the superscript-digit
          variants, in any case and with any extension, so "aux.c" counts),
          illegal characters (< > : " \ | ? * and control characters 0-31),
          trailing dots or spaces (Windows strips them), and names longer than
          255 characters.
          
  LENGTH  the full Windows path reaches MAX_PATH (260 characters including the
          terminating NUL), or for folders MAX_PATH - 12 (so an 8.3 filename
          can still be appended). Paths are measured as if the scanned
          directory lived at --dest (default: C:\<name of the scanned
          directory>), counting UTF-16 code units the way Windows does.
          
  ERROR   something went wrong while scanning: an unreadable directory, a
          scan target that isn't a directory, or a crash.
          

Output: one record per line, four TAB-separated fields:

    TAG <TAB> TYPE <TAB> PATH <TAB> DETAIL

  TYPE    dir, file, or link (a symlink); "-" when it doesn't apply.
  
  PATH    relative to the scanned directory. Control characters, including
          tab and newline, appear as \xNN so a record is always one line.
          
  DETAIL  CASE, MERGE: group=N. Entries with the same N collide with each other.
          NAME:        comma-separated reasons: reserved=NUL, illegal=<chars>,
                       trailing-dot, trailing-space, name-length=N
          LENGTH:      len=N,limit=N   (limit = longest allowed path)
          ERROR:       the error message

Lines starting with "#" are comments for people; parsers can skip them. The
last line is "# summary ..." with a count per tag (CASE and MERGE count
groups) and the assumptions used for path lengths.

Logging is off by default. With --log, every line printed to the console is
also appended, unchanged, to the file given by --error-log (default:
casecheck-errors.log in the current directory), so the two always match. The
file is only created if there is at least one problem or error; a clean run
with --log creates no file.

Exit status: 0 = no problems, 1 = problems found, 2 = error / incomplete scan.

Symlinks are never followed; they are treated as ordinary entries.

Installation:

Put the casecheck file somewhere in your path and do "chmod +x casecheck" to
the file to make it executable.
