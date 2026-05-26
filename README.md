# Lab: Search a String in Files and Save the Output — `grep`, `>`, `tee`

**Series:** linux-ops-mastery — RHCSA Essential Tools & File Operations
**Subjects covered:** Combining `grep` for content search with redirection (`>`, `>>`) and `tee` for capture; `grep -r` for recursive search; `-n` for line numbers; `-l` for matching filenames; `-c` for per-file counts; `-H` to force filename printing; `--include` / `--exclude` glob filters; `--color=auto` for highlight; writing artifacts that an exam grader can read; the standard "search → narrow → save → verify" pipeline
**Career arcs covered:** RHCSA ("search for the string X under /etc and save results to a file" — extremely common), RHCE (Ansible `lineinfile:` and `replace:` after locating lines), SRE (incident triage: "grep ERROR across /var/log and capture timestamps"), DevOps (log aggregation, CI failure search), AI/MLOps (search training logs for specific loss markers)
**Prerequisite:** Labs 01, 14
**Time Estimate:** 25 to 35 minutes
**Difficulty arc:** Task 1 foundation (`grep STRING file`) · 2 recursive search · 3 narrowing with flags · 4 saving with `>` / `tee` · 5 capturing both matches and counts · 6 RHCSA exam-realistic capstone

---

## Objective

Combine `grep` with redirection (`>`, `>>`) and `tee` to produce **artifacts** — files that a grader can read — instead of "answers" that scroll off the screen. By the end of this lab you can answer any "search for X and save results" prompt in two seconds and you know which flags narrow the noise.

The capstone is an exam-realistic prompt: *"Recursively search `/etc/` for the string `PermitRootLogin` and save (a) the matching lines with file paths and line numbers to `/root/permit-root-lines.txt`, (b) the list of unique files that contain a match to `/root/permit-root-files.txt`, and (c) the per-file match count to `/root/permit-root-counts.txt`."*

> **Lab safety note:** Every operation is read-only on `/etc/` and `/var/log/`. Outputs land in `/tmp/grep-lab` and `/root/`.

---

## Concept: `grep` Reads Streams; Redirection Captures Them

`grep` reads lines from files (or stdin) and writes matching lines to stdout. Combine with `>` and `tee` to **persist** the answer.

```
   ┌──────────────────────────────────────────┐
   │  grep PATTERN file        (or stdin)     │
   ├──────────────────────────────────────────┤
   │  stdout: matching lines                  │
   │  stderr: "no such file" / "Permission denied"│
   └──────────────────────────────────────────┘
          │              │
          │              └─ 2>/dev/null  to silence
          │
   ┌──────┴──────────────────────────────────┐
   │  > FILE      (capture, overwrite)        │
   │  >> FILE     (capture, append)           │
   │  | tee FILE  (capture AND display)       │
   └─────────────────────────────────────────┘
```

> **Why this matters:** A grader script looks at the file you produced. The fastest exam answer is *type the right grep, redirect to the right path, verify with `head`*. Practice it until it's reflex.

---

## 📜 Why the `grep + redirect` Pattern Exists — The Story

`grep` was created in **1973** when Ken Thompson distilled the "**g**lobal **r**egular **e**xpression **p**rint" function out of the `ed` editor into a standalone command. Almost the next day, somebody redirected its output to a file — because the whole point of Unix's "small tools that read stdin and write stdout" philosophy is that you can chain them together with `|`, `>`, `>>`.

The pattern `cmd | grep | tee FILE` is therefore as old as Unix itself. It is **the** representative example of the Unix philosophy: produce streams, filter streams, capture streams. Half a century later, it is still the most commonly-typed shell idiom on Earth.

> **The point of the story:** This lab is not about new tools. It is about combining the tools you already know — `grep`, `>`, `>>`, `tee` — into the artifact-producing pipelines a grader (or a real-world incident) can read.

---

## 👪 The Search-and-Save Family

### Grep flags you'll lean on

| Flag | Meaning |
|---|---|
| `-r` / `-R` | Recurse into directories |
| `-n` | Line numbers |
| `-H` | Always print filename (default with multiple files) |
| `-h` | Suppress filenames |
| `-l` | Print filenames with at least one match (no lines) |
| `-L` | Inverse of `-l` — files with NO matches |
| `-c` | Per-file match counts |
| `-i` | Case-insensitive |
| `-v` | Invert match |
| `-w` | Whole-word match |
| `-E` | Extended regex |
| `-F` | Fixed string (no regex interpretation) |
| `--include='*.conf'` | Glob filter when recursing |
| `--exclude-dir='.git'` | Skip directories |
| `-A N` / `-B N` / `-C N` | Lines after / before / around match |
| `--color=auto` | Highlight matches when output is a TTY |

### Output-capture sidekicks

| Pattern | Use case |
|---|---|
| `grep PAT FILE > out.txt` | Save matches (overwrite) |
| `grep PAT FILE >> out.txt` | Append matches |
| `grep PAT FILE \| tee out.txt` | Save AND display |
| `grep PAT FILE \| tee -a out.txt` | Append AND display |
| `grep PAT FILE 2>/dev/null > out.txt` | Silence noise, save matches |
| `grep -l PAT -r DIR > files.txt` | Save filenames with matches |
| `grep -c PAT -r DIR > counts.txt` | Save per-file counts |

> **The point of the family tree:** Pick `grep` flags by what you want in the output, then pair with the right redirect (`>`, `>>`, `tee`).

---

## 🔬 The Anatomy of `grep` Output — In One Diagram

```
$ grep -rn 'PermitRootLogin' /etc/ssh
/etc/ssh/sshd_config:36:#PermitRootLogin prohibit-password
/etc/ssh/sshd_config.d/50-redhat.conf:14:PermitRootLogin no
└────────────────────┘ │ └─────────────────────────────────────┘
                       │                  └─ Matching line content
                       └─ Line number (with -n)
filename
```

Default output with multiple input files: `file:lineno:content`. With `-h`, drop the filename; with `-H`, force it. With `-l`, only print the file.

> **Reading rule:** The three-part `file:line:content` shape is the entire `grep` audit trail. Train yourself to recognize and parse it instantly.

---

## 📚 Search & Save Reference Table

| Task | Command | Notes |
|---|---|---|
| Search one file | `grep PAT FILE` | Default |
| Search recursively | `grep -r PAT DIR` | Walks subdirs |
| With line numbers | `grep -rn PAT DIR` | Most common |
| Case-insensitive | `grep -i PAT FILE` | |
| Whole word | `grep -w PAT FILE` | `apt` does not match `adapt` |
| Inverse | `grep -v PAT FILE` | Lines NOT matching |
| List files only | `grep -rl PAT DIR` | One filename per match |
| Files with NO match | `grep -rL PAT DIR` | Inverse of `-l` |
| Count per file | `grep -rc PAT DIR` | Number-per-file |
| Total count | `grep -r PAT DIR \| wc -l` | Total across all files |
| Include glob | `grep -r --include='*.conf' PAT DIR` | Only `.conf` files |
| Exclude dir | `grep -r --exclude-dir='.git' PAT DIR` | Skip `.git/` |
| Context lines | `grep -C 2 PAT FILE` | 2 before + 2 after |
| Save matches | `grep -rn PAT DIR > out.txt` | Capture |
| Append matches | `grep -rn PAT DIR >> running.log` | Accumulate |
| Display + save | `grep PAT FILE \| tee out.txt` | Both |
| Silence permission noise | `grep -rn PAT /etc 2>/dev/null > out.txt` | Common |

> **Rule one of grep + redirect:** Pick the grep flags first; the redirect always goes at the end.

---

## 🎯 Career Pathway Sidebar

| Level | Why this lab matters |
|---|---|
| **RHCSA candidate** | Exam tasks frequently say "find lines containing X and save to /root/file." This is the pattern. |
| **RHCE candidate** | Ansible's `lineinfile:` and `replace:` modules let you ACT on what `grep -rn` finds. |
| **SRE / Platform** | `grep -rn ERROR /var/log 2>/dev/null > /tmp/incident.txt` is the entry-level on-call lookup. |
| **DevOps** | CI/CD pipelines: `grep -rn 'TEST FAILED' results/ \| tee fail-summary.txt`. |
| **AI / MLOps** | `grep -i 'val_loss' runs/exp-*/log.txt > metrics.txt` for cross-experiment summaries. |

---

## 🔧 The 6 Tasks

> Six exam-realistic phases that build the **grep → narrow → save → verify** habit.

---

### Task 1 — Single-file search; redirect with `>` and `tee`

**Purpose:** Search one file, save with `>`, save-and-display with `tee`.

```bash
mkdir -p /tmp/grep-lab && cd /tmp/grep-lab

grep 'root' /etc/passwd
grep 'root' /etc/passwd > root-lines.txt
cat root-lines.txt

grep 'nologin' /etc/passwd | tee nologin-lines.txt | head -n 3
wc -l nologin-lines.txt
```

**Human-Readable Breakdown:** Search `/etc/passwd` for `root` and save with `>`. Search for `nologin`, pipe through `tee` to save and still display.

**Reading it left to right:** `grep PAT FILE` reads FILE line by line and prints lines containing PAT. `>` captures stdout. `tee` clones the stream to a file and re-emits to stdout, so the next stage (`head -n 3`) still sees the data.

**The story:** This is the simplest version of the artifact-producing pattern. The grader does not care if you saw the lines on screen — they only check the file.

**Expected output:**

```text
root:x:0:0:root:/root:/bin/bash
operator:x:11:0:operator:/root:/sbin/nologin
root:x:0:0:root:/root:/bin/bash
operator:x:11:0:operator:/root:/sbin/nologin
bin:x:1:1:bin:/bin:/sbin/nologin
daemon:x:2:2:daemon:/sbin:/sbin/nologin
adm:x:3:4:adm:/var/adm:/sbin/nologin
19 nologin-lines.txt
```

**Switches**

| Token | Meaning |
|---|---|
| `grep PAT FILE` | Print matches |
| `> FILE` | Capture |
| `\| tee FILE` | Capture AND display |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Empty output file | No matches; verify with bare `grep` first |
| Display empty but file has content | You captured with `>` (no display by design); use `tee` |
| Trailing newline different | `grep` includes the line terminator |

---

### Task 2 — Recursive search across a directory

**Purpose:** Use `grep -r` to search every file in a directory tree.

```bash
grep -r 'PermitRootLogin' /etc/ssh 2>/dev/null
grep -rn 'PermitRootLogin' /etc/ssh 2>/dev/null
grep -rn 'PermitRootLogin' /etc/ssh 2>/dev/null > /tmp/grep-lab/permit-root.txt
wc -l /tmp/grep-lab/permit-root.txt
head -n 3 /tmp/grep-lab/permit-root.txt
```

**Human-Readable Breakdown:** Recursively search `/etc/ssh` for `PermitRootLogin`. Add `-n` for line numbers, save to a file, verify with `wc -l` and `head`.

**Reading it left to right:** `-r` walks subdirectories. `-n` prepends line numbers. `2>/dev/null` discards "permission denied" noise that you'd get on a deeper search.

**The story:** This combo (`-rn`) is the daily reflex. The output shape `file:line:content` is universal and easy to parse downstream.

**Expected output:**

```text
/etc/ssh/sshd_config:#PermitRootLogin prohibit-password
/etc/ssh/sshd_config.d/50-redhat.conf:PermitRootLogin no
/etc/ssh/sshd_config:36:#PermitRootLogin prohibit-password
/etc/ssh/sshd_config.d/50-redhat.conf:14:PermitRootLogin no
2 /tmp/grep-lab/permit-root.txt
/etc/ssh/sshd_config:36:#PermitRootLogin prohibit-password
/etc/ssh/sshd_config.d/50-redhat.conf:14:PermitRootLogin no
```

**Switches**

| Token | Meaning |
|---|---|
| `-r` | Recurse |
| `-n` | Line numbers |
| `2>/dev/null` | Silence permission errors |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `grep: /etc/ssh/...: Permission denied` | Add `2>/dev/null` or run as root |
| No matches but you expected some | Pattern case mismatch — add `-i` |
| Too much output | Add `--include='*.conf'` to narrow |

---

### Task 3 — Narrow with `--include`, `-i`, `-w`, context

**Purpose:** Add precision flags to keep noise out.

```bash
grep -rn --include='*.conf' 'PermitRootLogin' /etc 2>/dev/null
grep -rni 'permitrootlogin' /etc 2>/dev/null | head -n 5
grep -rnw 'apt' /etc/yum.repos.d /etc/dnf 2>/dev/null
grep -rnC 2 'PermitRootLogin' /etc/ssh 2>/dev/null
```

**Human-Readable Breakdown:** Limit recursive search to `*.conf` files; case-insensitive; whole-word; with 2 lines of context.

**Reading it left to right:** `--include` filters by filename glob. `-i` ignores case. `-w` matches only whole words. `-C N` shows N context lines above and below.

**The story:** Without `--include`, recursive search hits binaries and logs you don't want. Without `-w`, `apt` matches `adapt`. Without `-C`, you see only the matching line — sometimes the lines above are the answer you actually need.

**Expected output:**

```text
/etc/ssh/sshd_config.d/50-redhat.conf:14:PermitRootLogin no
/etc/ssh/sshd_config.d/50-redhat.conf:14:PermitRootLogin no
/etc/ssh/sshd_config:36:#PermitRootLogin prohibit-password
...
/etc/ssh/sshd_config-34-Match Group sftponly
/etc/ssh/sshd_config-35-#  ForceCommand internal-sftp
/etc/ssh/sshd_config:36:#PermitRootLogin prohibit-password
/etc/ssh/sshd_config-37-#  ChrootDirectory /home/%u
/etc/ssh/sshd_config-38-
```

**Switches**

| Token | Meaning |
|---|---|
| `--include='GLOB'` | Filename filter for `-r` |
| `--exclude='GLOB'` | Skip filename glob |
| `--exclude-dir='DIR'` | Skip directory |
| `-i` | Case-insensitive |
| `-w` | Whole-word |
| `-C N` | N context lines |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `--include` ignored | Without `-r`, `--include` has no effect |
| Whole-word missed real matches | The "word" boundary is non-alphanumeric — sometimes too strict |
| Context lines garbled | Multiple matches on adjacent lines — that's expected |

---

### Task 4 — Capture lists (`-l`) and counts (`-c`)

**Purpose:** Produce three different artifact shapes — matching lines, list of files, per-file counts.

```bash
grep -rl 'PermitRootLogin' /etc 2>/dev/null
grep -rl 'PermitRootLogin' /etc 2>/dev/null > /tmp/grep-lab/files-with-match.txt
grep -rL 'PermitRootLogin' /etc/ssh 2>/dev/null

grep -rc 'PermitRootLogin' /etc/ssh 2>/dev/null
grep -rc 'PermitRootLogin' /etc/ssh 2>/dev/null > /tmp/grep-lab/per-file-counts.txt

cat /tmp/grep-lab/files-with-match.txt
echo "---"
cat /tmp/grep-lab/per-file-counts.txt
```

**Human-Readable Breakdown:** `-l` lists filenames with at least one match. `-L` lists files with **no** matches. `-c` prints `file:count` (count of zero shown too — which can be noisy).

**Reading it left to right:** Three different artifact shapes produced from the same predicate. Each fits a different exam prompt: "list the files," "find files missing X," "count matches per file."

**The story:** Knowing which `-l` / `-L` / `-c` to reach for is the difference between answering in one command and chaining `grep | awk`.

**Expected output:**

```text
/etc/ssh/sshd_config
/etc/ssh/sshd_config.d/50-redhat.conf
... (other files without the match)
/etc/ssh/sshd_config:1
/etc/ssh/sshd_config.d/50-redhat.conf:1
... (and many files with :0 if `-c` mode)
/etc/ssh/sshd_config
/etc/ssh/sshd_config.d/50-redhat.conf
---
/etc/ssh/sshd_config:1
/etc/ssh/sshd_config.d/50-redhat.conf:1
...
```

**Switches**

| Token | Meaning |
|---|---|
| `-l` | List filenames with matches |
| `-L` | List filenames with NO matches |
| `-c` | Per-file count |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `-c` returned zeros | Normal — filter with `awk -F: '$NF>0'` |
| `-l` duplicates files | `-l` already de-dupes per file; verify with `sort -u` |
| `-L` returned huge list | All files without the pattern — narrow with `--include` |

---

### Task 5 — Combine matches + counts + tee for a single audit log

**Purpose:** Stack all three artifacts into one growing audit log using `tee -a`.

```bash
cd /tmp/grep-lab
> audit.log

{
  echo "===== matches ====="
  grep -rn 'PermitRootLogin' /etc/ssh 2>/dev/null
  echo
  echo "===== files with match ====="
  grep -rl 'PermitRootLogin' /etc 2>/dev/null
  echo
  echo "===== per-file counts ====="
  grep -rc 'PermitRootLogin' /etc/ssh 2>/dev/null | awk -F: '$NF>0'
} | tee audit.log | tail -n 5

wc -l audit.log
```

**Human-Readable Breakdown:** Use a `{ ... }` brace group with multiple commands; pipe the whole group through `tee` so the combined output lands in `audit.log` while still displaying on screen.

**Reading it left to right:** `{ cmd1; cmd2; }` runs commands sequentially in the current shell. Their combined stdout flows into `tee`. The `awk -F: '$NF>0'` removes per-file lines with count 0.

**The story:** Real-world audits look like this — one file with multiple labeled sections. `tee` makes building it a one-pass operation.

**Expected output:**

```text
===== files with match =====
/etc/ssh/sshd_config
/etc/ssh/sshd_config.d/50-redhat.conf

===== per-file counts =====
/etc/ssh/sshd_config:1
/etc/ssh/sshd_config.d/50-redhat.conf:1
15 audit.log
```

**Switches**

| Token | Meaning |
|---|---|
| `{ ...; ...; }` | Brace group — commands run in current shell |
| `awk -F: '$NF>0'` | Keep lines where last field > 0 |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Brace group syntax error | Spaces around `{` and `}` matter |
| Headings not in log | The `echo` lines must be inside the group |
| Heredoc instead of brace group | Heredoc is for stdin to one command; brace group runs multiple commands |

---

### Task 6 — Capstone: RHCSA-realistic three-artifact search

**Task statement:** *"Recursively search `/etc/` for the string `PermitRootLogin`. Save (a) matching lines with file paths and line numbers to `/root/permit-root-lines.txt`, (b) unique filenames containing a match to `/root/permit-root-files.txt`, and (c) per-file match counts (excluding zero) to `/root/permit-root-counts.txt`. Verify all three artifacts exist and report the totals."*

**Purpose:** Execute three saving patterns at once and verify.

```bash
sudo -i

grep -rn 'PermitRootLogin' /etc 2>/dev/null \
  > /root/permit-root-lines.txt

grep -rl 'PermitRootLogin' /etc 2>/dev/null \
  > /root/permit-root-files.txt

grep -rc 'PermitRootLogin' /etc 2>/dev/null \
  | awk -F: '$NF>0' \
  > /root/permit-root-counts.txt

# Verify
echo "lines:  $(wc -l < /root/permit-root-lines.txt)"
echo "files:  $(wc -l < /root/permit-root-files.txt)"
echo "counts: $(wc -l < /root/permit-root-counts.txt)"

head -n 3 /root/permit-root-lines.txt
echo
head -n 3 /root/permit-root-files.txt
echo
head -n 3 /root/permit-root-counts.txt

for f in /root/permit-root-lines.txt /root/permit-root-files.txt /root/permit-root-counts.txt; do
  test -s "$f" && echo "VERIFY: $f exists and is non-empty"
done
```

**Human-Readable Breakdown:** Become root. Run three separate `grep` calls — one per artifact shape — saving each to its own file. Verify all three are non-empty, count lines, and print heads.

**Layer stack you built:**

```text
/etc
   │
   │  grep -rn 'PermitRootLogin' /etc
   │
   ├── /root/permit-root-lines.txt    (file:line:content)
   ├── /root/permit-root-files.txt    (filenames, deduped)
   └── /root/permit-root-counts.txt   (file:count, count>0)
```

**The story:** This is the **canonical exam pattern** when a prompt asks for multiple shapes of the same search. Memorize the spine: separate `grep` per shape; each redirect goes to its own file.

**Expected verification output:**

```text
lines:  2
files:  2
counts: 2
/etc/ssh/sshd_config:36:#PermitRootLogin prohibit-password
/etc/ssh/sshd_config.d/50-redhat.conf:14:PermitRootLogin no

/etc/ssh/sshd_config
/etc/ssh/sshd_config.d/50-redhat.conf

/etc/ssh/sshd_config:1
/etc/ssh/sshd_config.d/50-redhat.conf:1
VERIFY: /root/permit-root-lines.txt exists and is non-empty
VERIFY: /root/permit-root-files.txt exists and is non-empty
VERIFY: /root/permit-root-counts.txt exists and is non-empty
```

**Cleanup**

```bash
rm -rf /tmp/grep-lab
rm -f /root/permit-root-lines.txt /root/permit-root-files.txt /root/permit-root-counts.txt
exit
```

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `permit-root-counts.txt` huge | Forgot `awk -F: '$NF>0'` |
| `lines.txt` has line numbers but `files.txt` does too | You used `-rn` for both — re-run with `-rl` for files |
| Empty files | Pattern wrong; verify with bare grep |

---

## 🔍 Search-and-Save Decision Guide

```
What artifact do you need?
  │
  ├── "Lines with location"
  │       └── ✅ grep -rn PAT PATH > lines.txt
  │
  ├── "Filenames only"
  │       └── ✅ grep -rl PAT PATH > files.txt
  │
  ├── "Filenames WITHOUT matches"
  │       └── ✅ grep -rL PAT PATH > files-missing.txt
  │
  ├── "Per-file count (>0)"
  │       └── ✅ grep -rc PAT PATH | awk -F: '$NF>0' > counts.txt
  │
  ├── "Total count"
  │       └── ✅ grep -r PAT PATH | wc -l
  │
  ├── "Need to see it scroll AND keep a copy"
  │       └── ✅ grep -rn PAT PATH | tee lines.txt
  │
  └── "Build a multi-section audit log"
          └── ✅ { echo HEADER; grep ...; echo HEADER2; grep ...; } | tee audit.log
```

---

## ✅ Lab Checklist (6 Tasks)

- [ ] 01 Search single file; save with `>` and `tee`
- [ ] 02 Recursive search; save lines artifact
- [ ] 03 Narrow with `--include`, `-i`, `-w`, `-C`
- [ ] 04 Save filenames with `-l`, counts with `-c`
- [ ] 05 Build a multi-section audit log via brace group + `tee`
- [ ] 06 Execute the RHCSA capstone — three artifact files for `PermitRootLogin`

---

## ⚠️ Common Pitfalls

| Mistake | Symptom | Fix |
|---|---|---|
| Forgot `-n` | No line numbers in output | Add `-n` |
| Forgot `-r` | Searched only the directory entry | Add `-r` |
| Forgot `2>/dev/null` | Permission noise in artifact | Append it |
| Used `>` when `tee` was needed | Lost display | Use `tee` |
| Wrong glob in `--include` | Quoted, missed matches | Quote literally: `--include='*.conf'` |
| `-c` reported zeros | Filter with `awk -F: '$NF>0'` |
| Wanted to grep stdin | `grep PAT FILE` runs against file; for stdin use `... \| grep PAT` |
| Pattern is a regex but used `-F` | Literal-only match | Drop `-F` |
| Mixed `grep -rn` and `grep -rl` artifacts | Different shapes | Run separate `grep` per artifact |
| Forgot to verify | Empty file shipped | Always `wc -l`/`head` after |

---

## 🎯 Career & Interview Strategy

**RHCSA candidate**
- Drill the three artifact shapes (`-rn`, `-rl`, `-rc`). Drill the three redirects (`>`, `>>`, `tee`). Combine on demand.

**RHCE candidate**
- Ansible `lineinfile:` + `regexp:` lets you both find and modify; for read-only audits, `command: grep -rn ...` with `register:`.

**SRE / Platform interview**
- "How would you capture every ERROR line across `/var/log` during an incident window?" → `grep -rn ERROR /var/log 2>/dev/null > /tmp/incident-errors-$(date +%s).txt`.

**DevOps**
- CI failure capture: `grep -rn 'TEST FAILED' results/ | tee fail-summary.txt`.

**AI / MLOps**
- `grep -i val_loss runs/exp-*/log.txt > metrics.txt` for cross-experiment summaries.

---

## 🔗 Related Labs

| Lab | Connection |
|---|---|
| Lab 01 — stdout Redirection | The `>` / `>>` / `tee` foundation |
| Lab 03 — Pipe Text Streams | The `|` foundation for combining `grep` with other tools |
| Lab 14 — Searching with `find` | When you need to find files BY PATH, not by content |
| Lab 22 — Filtering Text with `grep` and Regex | Deep dive on grep flags and patterns |
| Lab 25 — Extracting Columns with `awk` | Post-process `grep -c` counts |

---

## 👤 Author

**Kelvin R. Tobias**
[kelvinintech.com](https://kelvinintech.com) · [GitHub](https://github.com/kelvintechnical) · [LinkedIn](https://www.linkedin.com/in/kelvin-r-tobias-211949219)
