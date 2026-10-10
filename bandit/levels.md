# OverTheWire Bandit — Progress Log

**Rule:** passwords are never committed. Only the technique and reasoning.

Connection: `ssh banditN@bandit.labs.overthewire.org -p 2220`

## Level 0 → 1

**Goal:** password is in a file called `readme` in the home directory.

**Command:**
```bash
ls
cat readme
```

**Reasoning:** `ls` confirmed the file exists; the goal named it explicitly, so
`cat` on it was the direct path.

**Lesson:** The first level is intentionally trivial. It establishes the SSH
pattern (`-p 2220` for the non-standard port) and the workflow of "look, then read."

## Level 1 → 2

**Goal:** password is in a file called `-`.

**Command:**
```bash
ls -la
cat ./-
```

**Reasoning:** `cat -` hangs because `-` is a Unix convention meaning "read from
standard input." The `./` prefix forces `cat` to treat `-` as a path.

**Lesson:** Filenames can collide with tool conventions. Prefix with `./` to
disambiguate. This same class of bug appears in real command-injection and
argument-injection vulnerabilities.

## Level 2 → 3

**Goal:** password is in a file called `spaces in this filename`.

**Command:**
```bash
ls -la
cat "spaces in this filename"
```

**Reasoning:** Unquoted, the shell splits the name into four arguments (`spaces`,
`in`, `this`, `filename`). Quoting preserves it as a single argument.

**Lesson:** Shell word splitting. Unquoted variables in scripts are the root of
many injection bugs. `rm $FILE` is dangerous; `rm "$FILE"` is safe.

## Level 3 → 4

**Goal:** password is in a hidden file inside the `inhere` directory.

**Command:**
```bash
cd inhere
ls -la
cat .hidden
```

**Reasoning:** `ls` without `-a` skips files whose names start with `.`. The `-a`
flag reveals them.

**Lesson:** Hidden != secure. `.hidden` files are named by convention only. In
real systems, secrets in dotfiles (`.env`, `.aws/credentials`, `.git-credentials`)
are frequently overlooked.

## Level 4 → 5

**Goal:** find a human-readable file among many.

**Command:**
```bash
file ./*
cat ./-file07
```

**Reasoning:** `file` classifies each file by content type. `ASCII text` marks the
one to read.

**Lesson:** Content-based identification beats name-based. The `file` command reads
magic bytes, not extensions — the same principle behind modern file-identification
tooling in forensics.

## Level 5 → 6

**Goal:** a file with size 1033 bytes, human-readable, not executable.

**Command:**
```bash
find . -type f -size 1033c ! -executable
cat ./maybehere07/.file2
```

**Reasoning:** `find` with predicates narrows the search. `-size 1033c` means
"exactly 1033 bytes" (the `c` suffix means bytes, not blocks). `!` negates the
next predicate.

**Lesson:** `find` is the single most powerful tool for filesystem reconnaissance.
Every engagement starts with questions like "what files exist," "what's new,"
"what's world-writable," "what's setuid." `find` answers all of them.

## Level 6 → 7

**Goal:** a file somewhere on the server, owned by `bandit7`, group `bandit6`,
size 33 bytes.

**Command:**
```bash
find / -user bandit7 -group bandit6 -size 33c 2>/dev/null
```

**Reasoning:** `find /` starts at root. `2>/dev/null` discards permission errors
from directories the user can't read — otherwise the output is drowned in noise.

**Lesson:** Redirecting stderr is essential when searching filesystems you don't
own. The `2>&1` and `2>/dev/null` idioms show up constantly in tooling and exploit
development.

## Level 7 → 8

**Goal:** password is on the line containing the word `millionth` in `data.txt`.

**Command:**
```bash
grep "millionth" data.txt
```

**Reasoning:** `grep` filters lines by pattern. `data.txt` is far too large to
read directly.

**Lesson:** `grep` is the most-used text tool in security work — log analysis,
credential hunting, pattern matching. `cat` doesn't scale; `grep` does.
