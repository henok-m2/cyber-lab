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

## Level 8 → 9

**Goal:** password is the only line that occurs once in `data.txt`.

**Command:**
```bash
sort data.txt | uniq -u
```

**Reasoning:** `sort` groups identical lines together; `uniq -u` prints only lines
that appear exactly once. Without `sort`, `uniq` only compares adjacent lines.

**Lesson:** `uniq` requires sorted input. `sort | uniq -c` is the idiom for
frequency counting — used for log analysis (`sort | uniq -c | sort -rn`).

## Level 9 → 10

**Goal:** password is in `data.txt`, preceded by several `=` characters.

**Command:**
```bash
strings data.txt | grep "==="
```

**Reasoning:** `data.txt` is binary. `strings` extracts printable sequences;
`grep` filters for the marker described in the goal.

**Lesson:** `strings` is a workhorse in binary analysis — finding embedded paths,
URLs, and credentials in compiled binaries or firmware images.

## Level 10 → 11

**Goal:** password is in `data.txt`, base64-encoded.

**Command:**
```bash
base64 -d data.txt
```

**Reasoning:** The goal said base64. `-d` decodes.

**Lesson:** Base64 is an encoding, not encryption. It's used everywhere in web
tokens (JWTs), HTTP Basic auth, and encoded payloads. Recognizing the
`[A-Za-z0-9+/=]` alphabet is a core skill.

## Level 11 → 12

**Goal:** password is in `data.txt`, all letters ROT13'd.

**Command:**
```bash
cat data.txt | tr 'A-Za-z' 'N-ZA-Mn-za-m'
```

**Reasoning:** ROT13 rotates each letter 13 positions. `tr` maps the two halves
of the alphabet onto each other. Since 13 is half of 26, ROT13 is its own inverse.

**Lesson:** `tr` is a stream translator. ROT13 is trivially reversible and
provides zero security — it's an obfuscation, not encryption. Distinguishing the
two is fundamental.

## Level 12 → 13

**Goal:** password is in `data.txt`, which is a hexdump of a file that has been
repeatedly compressed.

**Command:**
```bash
mkdir /tmp/bandit12 && cd /tmp/bandit12
cp ~/data.txt .
xxd -r data.txt > data.bin
file data.bin        # identify: gzip, bzip2, tar, etc.
mv data.bin data.gz
gzip -d data.gz      # repeat with bzip2, tar, etc. per `file` output
```

**Reasoning:** `xxd -r` reverses the hex dump into binary. `file` identifies the
compression format; decompress and repeat. Work in `/tmp` — you can't write to
the home directory.

**Lesson:** Layered compression is used in malware packing and exfiltration.
The loop "identify format → decompress → check again" is the exact workflow of
unpacking packed binaries in malware analysis.

## Level 13 → 14

**Goal:** use the SSH private key at `sshkey.private` to log in as `bandit14`.

**Command:**
```bash
chmod 600 sshkey.private
ssh -i sshkey.private bandit14@localhost -p 2220
```

**Reasoning:** SSH refuses keys with loose permissions. `chmod 600` fixes that.
`-i` specifies the identity file. `localhost` because you're already on the
bandit server.

**Lesson:** SSH key authentication. This is how production access works —
passwords are the exception, keys are the rule. Understanding `chmod 600` on
keys is a routine part of Linux ops.

## Level 14 → 15

**Goal:** submit the current level's password to port 30000 on localhost.

**Command:**
```bash
cat /etc/bandit_pass/bandit14 | nc localhost 30000
```

**Reasoning:** The previous password is stored in `/etc/bandit_pass/bandit14`.
`nc` pipes it to the service on port 30000, which replies with the next password.

**Lesson:** `nc` as a client for arbitrary TCP services. You'll use it for
banner grabbing, manual protocol interaction, and as a listener in reverse-shell
workflows.

## Level 15 → 16

**Goal:** submit the password to port 30001, which requires SSL/TLS.

**Command:**
```bash
cat /etc/bandit_pass/bandit15 | openssl s_client -connect localhost:30001 -quiet
```

**Reasoning:** `openssl s_client` establishes a TLS session and passes stdin
through to the encrypted socket.

**Lesson:** `openssl s_client` is the TLS equivalent of `nc`. Use it to inspect
certificates, verify TLS configuration, and interact with any TLS-protected
service manually.
