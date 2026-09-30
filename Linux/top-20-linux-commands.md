# Top 20 Linux Commands Everyone Should Know

## Brief

The Linux terminal feels scary at first, but day to day you only need about **20 commands**. They cover moving around the filesystem, working with files, reading and searching text, checking processes, handling permissions, and talking to the network.

Learn these well and you can get around any Linux server, Docker container, or CI box.

---

## The analogy that makes it click

**The shell is a conversation with the computer, and every command is a verb.**

```
command   -flags          arguments
  ls       -la            /var/log
  verb     how to do it   what to do it to
```

- The **command** says *what* to do (`ls` = list).
- The **flags** (options) say *how* (`-l` = long format, `-a` = include hidden files).
- The **arguments** say *what to act on* (`/var/log`).

A few conventions hold for almost every command:

- Short flags use one dash and can be combined: `ls -l -a -h` is the same as `ls -lah`.
- Long flags use two dashes and are spelled out: `ls --all`, `rm --recursive`.
- `--` means "no more flags after this", so `rm -- -weird-name.txt` deletes a file whose name starts with a dash.
- When you get stuck, `man <command>` opens the full manual (`q` to quit), and `<command> --help` prints a short summary.

---

## Quick reference

| # | Command | What it does | Category |
|---|---|---|---|
| 1 | `pwd` | Print the current directory | Navigation |
| 2 | `ls` | List files | Navigation |
| 3 | `cd` | Change directory | Navigation |
| 4 | `mkdir` | Make a directory | Files |
| 5 | `touch` | Create an empty file or update its timestamp | Files |
| 6 | `cp` | Copy files and directories | Files |
| 7 | `mv` | Move or rename | Files |
| 8 | `rm` | Remove files and directories | Files |
| 9 | `cat` | Print file contents | Viewing |
| 10 | `less` | Scroll through a file one page at a time | Viewing |
| 11 | `head` / `tail` | Show the start or end of a file | Viewing |
| 12 | `grep` | Search text for a pattern | Searching |
| 13 | `find` | Find files by name, size, time… | Searching |
| 14 | `chmod` | Change permissions | Permissions |
| 15 | `chown` | Change owner | Permissions |
| 16 | `sudo` | Run a command as root | Permissions |
| 17 | `ps` / `top` | See running processes | Processes |
| 18 | `kill` | Stop a process | Processes |
| 19 | `df` / `du` | Disk space used and free | System |
| 20 | `curl` | Make HTTP requests | Network |

---

## 1. Navigation

### 1. `pwd`: where am I?

**P**rint **W**orking **D**irectory. Every shell session has a "current directory", and relative paths like `./src` or `../config` are worked out from it. `pwd` prints the full path of that directory.

```bash
$ pwd
/home/harsh/projects
```

**Options**

| Flag | Meaning |
|---|---|
| `-L` | Logical path: keep symlinks as you followed them (the default) |
| `-P` | Physical path: resolve symlinks to the real location on disk |

```bash
# /var/www is a symlink to /srv/www
$ cd /var/www
$ pwd          # /var/www
$ pwd -P       # /srv/www
```

**When you'll use it**

- You've done a string of `cd`s and lost track of where you are.
- In scripts: `SCRIPT_DIR=$(pwd)` saves the directory so you can come back to it later.
- Before running anything destructive (`rm -rf *`), check `pwd` so you know which directory it will hit.

**Tip:** the variable `$PWD` holds the same value, so `echo $PWD` works too.

---

### 2. `ls`: what's here?

**L**i**s**t lists the contents of a directory. With no argument it lists the current directory. You can also pass one or more paths or globs.

```bash
ls                  # current directory
ls /etc             # a specific directory
ls *.js             # only .js files here
ls src tests        # two directories at once
```

**Common options**

| Flag | Meaning |
|---|---|
| `-l` | Long format: permissions, links, owner, group, size, date, name |
| `-a` | Show **all** files, including hidden ones (names starting with `.`) |
| `-A` | Like `-a` but skips `.` and `..` |
| `-h` | Human-readable sizes (`4.0K`, `12M`, `1.2G`), used with `-l` |
| `-t` | Sort by modification time, newest first |
| `-S` | Sort by size, biggest first |
| `-r` | Reverse the sort order |
| `-R` | List subdirectories recursively |
| `-d` | List the directory itself, not what's inside it |
| `-1` | One entry per line (handy when piping into other commands) |

**Reading `ls -l` output**

```
-rw-r--r--  1  harsh  harsh  2048  Oct 1 10:00  notes.md
│└──┬────┘  │  └─┬─┘  └─┬─┘  └┬─┘  └────┬────┘  └──┬───┘
│ perms   links owner  group  size   modified     name
└─ type: - file, d directory, l symlink
```

**Everyday combinations**

```bash
ls -lah             # everything, long format, readable sizes: the one to remember
ls -lt | head       # the 10 most recently changed files
ls -lS              # find what's taking up space in this folder
ls -ltr             # oldest first, newest at the bottom (good for log folders)
ls -d */            # only directories
ls -la ~            # see your dotfiles (.bashrc, .ssh, .gitconfig)
```

**Gotchas**

- Hidden files (`.env`, `.git`) **don't show up** without `-a`. If a config file seems to be missing, try `ls -a`.
- `ls -l` on a directory shows its size as `4.0K`, which is the size of the directory entry, not what's inside it. Use `du -sh` (command 19) to see a folder's real size.
- Don't parse `ls` output in scripts, because filenames with spaces break it. Use `find` or shell globs instead.

---

### 3. `cd`: go somewhere

**C**hange **D**irectory moves you to another directory. It's built into the shell rather than being a separate program, because it has to change the shell's own state.

```bash
cd /var/log     # absolute path (starts at root /)
cd projects     # relative path (from where you are)
cd ..           # up one level
cd ../..        # up two levels
cd ~            # home directory
cd              # also home directory
cd -            # back to the previous directory (toggles between two)
cd ~/Desktop    # a path inside your home
```

**Path shortcuts**

| Symbol | Meaning |
|---|---|
| `/` | Root of the whole filesystem |
| `~` | Your home directory (`/home/<you>`) |
| `~bob` | Another user's home directory |
| `.` | Current directory |
| `..` | Parent directory |
| `-` | Previous directory (only with `cd`) |

**Absolute vs relative paths**

```
Absolute: /home/harsh/projects/app    → starts with /, works from anywhere
Relative: projects/app                → worked out from the current directory
```

**Tips**

- Press `Tab` to autocomplete directory names. Press it twice to see every option.
- Quote paths that contain spaces: `cd "My Documents"` or `cd My\ Documents`.
- `cd -` is great for switching back and forth between two directories, e.g. your code and your logs.
- `pushd /some/dir` and `popd` work like `cd` but keep a stack of directories, so you can return to earlier ones in order.

---

## 2. Working with files

### 4. `mkdir`: make a directory

**M**a**k**e **dir**ectory creates one or more new directories.

```bash
mkdir logs
mkdir logs cache tmp          # several at once
```

**Common options**

| Flag | Meaning |
|---|---|
| `-p` | Create any missing **parent** directories, and don't complain if the directory already exists |
| `-v` | Verbose: print each directory as it's created |
| `-m` | Set permissions at creation, e.g. `-m 700` |

```bash
mkdir -p app/src/utils        # creates app, app/src and app/src/utils in one go
mkdir -p backups/2026/10      # safe to run again and again
mkdir -m 700 secrets          # only you can read, write or enter it
```

**Brace expansion trick**

The shell expands `{a,b,c}` before `mkdir` runs, so you can build a whole project tree in one line:

```bash
mkdir -p project/{src,tests,docs}
mkdir -p project/src/{components,hooks,utils}
```

**Gotcha:** without `-p`, `mkdir a/b/c` fails if `a/b` doesn't exist yet, and `mkdir logs` fails if `logs` already exists. In scripts, just use `-p` every time.

---

### 5. `touch`: create an empty file

`touch` has two jobs:

1. If the file **doesn't exist**, create it as an empty file.
2. If the file **does exist**, update its "last modified" and "last accessed" timestamps to now, without changing its contents.

```bash
touch index.js                # create an empty file
touch a.txt b.txt c.txt       # several at once
touch src/{app,server}.js     # brace expansion works here too
```

**Common options**

| Flag | Meaning |
|---|---|
| `-c` | Don't create the file if it doesn't exist, only update timestamps |
| `-m` | Only change the modification time |
| `-a` | Only change the access time |
| `-d` | Set a specific date instead of now |
| `-r` | Copy the timestamps from another file |

```bash
touch -d "2026-01-01 10:00" report.txt    # backdate a file
touch -r original.txt copy.txt            # give copy.txt the same timestamps as original.txt
```

**When you'll use it**

- Quickly creating placeholder files: `.gitkeep`, `.env`, `README.md`.
- Making build tools like `make` think a file changed so they rebuild it.
- Creating a "marker" file that a script checks for, e.g. `touch /tmp/deploy.lock`.

**Gotcha:** `touch` needs the parent directory to already exist. `touch a/b/file.txt` fails unless `a/b` is there, so run `mkdir -p a/b` first.

---

### 6. `cp`: copy

**C**o**p**y copies files or directories from a source to a destination.

```bash
cp source destination
cp file.txt backup.txt            # copy to a new name
cp file.txt ~/Documents/          # copy into a directory, same name
cp a.txt b.txt c.txt dest/        # several files into a directory (last argument = destination)
cp *.jpg photos/                  # glob
```

**Common options**

| Flag | Meaning |
|---|---|
| `-r` / `-R` | Recursive: required for copying directories |
| `-i` | Interactive: ask before overwriting |
| `-n` | No-clobber: never overwrite existing files |
| `-u` | Update: only copy if the source is newer than the destination |
| `-v` | Verbose: print each file as it's copied |
| `-p` | Preserve permissions, owner and timestamps |
| `-a` | Archive: `-r` plus preserve everything (and keep symlinks as symlinks). Best for backups |

```bash
cp -r src/ src_backup/            # copy a whole directory
cp -a /var/www /backup/www        # exact copy with permissions and timestamps kept
cp -iv config.yml config.yml.bak  # ask before overwriting, show what happened
cp -u *.md docs/                  # only copy files that changed
```

**Gotchas**

- **`cp` overwrites silently by default.** Copying onto an existing file replaces it with no warning. Use `-i` or `-n` when that matters.
- Copying a directory without `-r` gives `cp: -r not specified; omitting directory`.
- A trailing slash can matter. `cp -r src dest` creates `dest/src` if `dest` already exists, or creates `dest` as the copy if it doesn't. Check the result with `ls` when it matters.
- For large copies or copies to another machine, `rsync -av src/ dest/` is usually better: it can resume and it only copies what changed.

---

### 7. `mv`: move or rename

**M**o**v**e moves files and directories to a new location. Renaming is just moving to a new name in the same place, so there's no separate rename command.

```bash
mv old.txt new.txt                # rename
mv file.txt ~/Documents/          # move into another directory
mv file.txt ~/Documents/new.txt   # move and rename at once
mv *.log logs/                    # move many files
mv project/ archive/project-2025/ # move or rename a directory (no -r needed)
```

**Common options**

| Flag | Meaning |
|---|---|
| `-i` | Ask before overwriting |
| `-n` | Never overwrite |
| `-u` | Only move if the source is newer |
| `-v` | Print what's being moved |
| `-b` | Make a backup of any file that would be overwritten |

**How it works under the hood**

- Moving **within the same disk** only changes the file's name and location in the filesystem, so it's instant even for a 50 GB file.
- Moving **to a different disk or partition** has to copy the data and then delete the original, so it takes time.

**Gotchas**

- Like `cp`, `mv` **overwrites silently**. `mv a.txt b.txt` destroys the old `b.txt`. Use `-i` to be safe.
- If the destination is an existing directory, the source goes *inside* it. If it isn't, the source is renamed to that name. `mv logs backup` behaves differently depending on whether `backup/` exists.
- To rename many files at once (e.g. every `.jpeg` to `.jpg`), use a loop:

```bash
for f in *.jpeg; do mv "$f" "${f%.jpeg}.jpg"; done
```

---

### 8. `rm`: delete

**R**e**m**ove deletes files, and directories with `-r`.

```bash
rm file.txt
rm a.txt b.txt
rm *.tmp
rm -r folder/
```

**Common options**

| Flag | Meaning |
|---|---|
| `-r` / `-R` | Recursive: delete a directory and everything inside it |
| `-f` | Force: skip prompts, ignore files that don't exist |
| `-i` | Ask before every file |
| `-I` | Ask once before deleting more than 3 files or deleting recursively (less annoying than `-i`) |
| `-v` | Print each file as it's deleted |
| `-d` | Delete an empty directory (same as `rmdir`) |

```bash
rm -i important.txt           # ask first
rm -rI node_modules/          # one confirmation for the whole tree
rm -rf build/ dist/           # clear build output, no questions asked
rmdir empty_folder            # only works if the folder is empty (a safe way to delete)
```

> ⚠️ **There is no recycle bin.** `rm` deletes for good, and recovering files afterwards is hard and often impossible.

**Safety habits**

- Run `ls` with the same pattern first to see what will be deleted: `ls *.log`, then `rm *.log`.
- Never run `rm -rf /` or `rm -rf ~`. Modern systems block `rm -rf /`, but not `rm -rf /*`.
- Watch out for empty variables: `rm -rf $DIR/` becomes `rm -rf /` if `$DIR` is unset. In scripts, use `rm -rf "${DIR:?}/"`, which stops with an error if `DIR` is empty.
- Watch out for stray spaces: `rm -rf ./ build` (with a space) deletes the **current directory** as well as `build`.
- On a desktop, `gio trash file.txt` (or the `trash-cli` package) sends files to the trash instead.

---

## 3. Viewing files

### 9. `cat`: print a whole file

`cat` is short for con**cat**enate. It reads one or more files and prints them one after another to the terminal. Most of the time it's used to dump a single small file.

```bash
cat config.yml                # print a file
cat a.txt b.txt               # print two files one after the other
cat a.txt b.txt > both.txt    # join them into a new file
cat part1 part2 >> all.txt    # append to an existing file
```

**Common options**

| Flag | Meaning |
|---|---|
| `-n` | Number every line |
| `-b` | Number only non-empty lines |
| `-A` | Show hidden characters: tabs as `^I`, line ends as `$`, Windows line endings as `^M` |
| `-s` | Squeeze several blank lines into one |

```bash
cat -n app.js                 # with line numbers
cat -A script.sh              # find why a script breaks: often ^M (Windows line endings)
```

**Quick file creation with a heredoc**

```bash
cat > .env <<EOF
PORT=3000
NODE_ENV=development
EOF
```

**Related commands**

- `tac`: prints a file with the lines in reverse order (last line first).
- `wc -l file`: counts lines instead of printing them.
- `bat`: a modern `cat` with syntax highlighting (if installed).

**Gotchas**

- Don't `cat` a huge file. It floods the terminal. Use `less`, `head` or `tail` instead.
- Don't `cat` binary files (images, executables). They print garbage and can mess up your terminal. If that happens, type `reset`.
- `cat file | grep x` works, but `grep x file` does the same job without the extra process.

---

### 10. `less`: page through a file

`less` opens a file in a scrollable viewer. It doesn't load the whole file into memory first, so it opens multi-gigabyte logs instantly. (It replaced an older tool called `more`, hence the joke "less is more".)

```bash
less /var/log/syslog
less +G app.log               # open at the end of the file
less -N app.js                # show line numbers
less -S wide.csv              # don't wrap long lines; scroll sideways with arrow keys
ps aux | less                 # page through any command's output
```

**Keys inside `less`**

| Key | Action |
|---|---|
| `Space` / `f` | Page down |
| `b` | Page up |
| `j` / `k` or `↓` / `↑` | One line down / up |
| `g` / `G` | Jump to start / end |
| `50g` | Jump to line 50 |
| `/word` | Search forward |
| `?word` | Search backward |
| `n` / `N` | Next / previous search match |
| `&pattern` | Show only lines matching the pattern (press `&` then Enter to clear) |
| `F` | Follow mode: like `tail -f`. `Ctrl+C` to stop following |
| `-N` (while open) | Toggle line numbers |
| `h` | Help |
| `q` | Quit |

**Why `less` instead of an editor?**

- It's read-only, so you can't change a config or log by accident.
- It's fast on huge files where editors struggle.
- It's the default viewer for `man` pages and `git log`, so the same keys work there.

**Tip:** `less -R` shows colored output properly, e.g. `grep --color=always error app.log | less -R`.

---

### 11. `head` and `tail`: start or end of a file

`head` prints the **first** lines of a file and `tail` prints the **last** lines. Both default to 10 lines.

```bash
head file.txt                 # first 10 lines
head -n 20 file.txt           # first 20 lines
head -n -5 file.txt           # everything except the last 5 lines
head -c 100 file.bin          # first 100 bytes

tail file.txt                 # last 10 lines
tail -n 50 app.log            # last 50 lines
tail -n +2 data.csv           # everything from line 2 on (skip the CSV header)
```

**`tail` options**

| Flag | Meaning |
|---|---|
| `-n N` | Last N lines |
| `-n +N` | From line N to the end |
| `-f` | **Follow**: keep the file open and print new lines as they're written |
| `-F` | Follow, and keep going if the file is rotated or recreated (better for real logs) |
| `-q` | Don't print filename headers when tailing several files |

**Watching logs live**

```bash
tail -f app.log                         # watch one log
tail -F /var/log/nginx/access.log       # survives log rotation
tail -f app.log | grep -i error         # watch for errors only
tail -f a.log b.log                     # watch two logs at once, with headers
```

Press `Ctrl+C` to stop.

**Other handy uses**

```bash
head -1 data.csv                        # just see the column names
ls -t | head -5                         # 5 most recently modified files
history | tail -20                      # your last 20 commands
sed -n '100,120p' file.txt              # a range from the middle: lines 100–120
```

**Tip:** for systemd services, `journalctl -u nginx -f` is the equivalent of `tail -f`.

---

## 4. Searching

### 12. `grep`: search inside files

`grep` (from **g**lobal **r**egular **e**xpression **p**rint) searches text for lines that match a pattern and prints those lines. It can search files, whole directory trees, or the output of another command.

```bash
grep "pattern" file
grep "error" app.log
ps aux | grep nginx
```

**Common options**

| Flag | Meaning |
|---|---|
| `-i` | Ignore case |
| `-r` / `-R` | Search directories recursively |
| `-n` | Show line numbers |
| `-v` | Invert: show lines that **don't** match |
| `-c` | Count matching lines instead of printing them |
| `-l` | Print only the names of files that match |
| `-w` | Match whole words only (`log` won't match `login`) |
| `-x` | Match whole lines only |
| `-o` | Print only the matching part, not the whole line |
| `-E` | Extended regex (`\|`, `+`, `?`, `{}` work without backslashes) |
| `-F` | Fixed string: no regex, the pattern is taken literally (faster, safe for special characters) |
| `-A N` / `-B N` / `-C N` | Show N lines **A**fter / **B**efore / around (**C**ontext) each match |
| `--include` / `--exclude` | Only search / skip files matching a glob |
| `--color` | Highlight matches |

**Examples**

```bash
grep -i "error" app.log                     # Error, ERROR, error...
grep -rn "TODO" src/                        # every TODO in the codebase, with file:line
grep -rl "API_KEY" .                        # which files mention API_KEY?
grep -v "DEBUG" app.log                     # hide the noise
grep -c "404" access.log                    # how many 404s?
grep -w "id" schema.sql                     # "id" as a word, not "user_id"
grep -C 3 "Exception" app.log               # each exception with 3 lines of context
grep -E "error|warn|fatal" app.log          # any of several words
grep -oE "[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+" access.log   # pull out IP addresses
grep -r "useState" src/ --include="*.tsx"   # only search .tsx files
grep -rn "console.log" . --exclude-dir=node_modules
```

**A quick regex cheat sheet**

| Pattern | Matches |
|---|---|
| `^abc` | Lines starting with `abc` |
| `abc$` | Lines ending with `abc` |
| `.` | Any single character |
| `a*` | Zero or more `a` |
| `[0-9]` | Any digit |
| `[^a-z]` | Any character that's not a lowercase letter |
| `a+`, `a?`, `a{2,4}` | One or more, optional, 2–4 times (with `-E`) |

**Gotchas**

- `ps aux | grep node` also matches the `grep node` process itself. Use `pgrep -a node` instead, or the trick `grep [n]ode`.
- Quote your patterns so the shell doesn't expand `*` or `$` before `grep` sees them: `grep "user.*id"`.
- `grep` exits with code 1 when nothing matches. That's useful in scripts (`if grep -q ...`) but can trip up `set -e`.
- On large codebases, `rg` (ripgrep) does the same job much faster and skips `.gitignore`d files automatically.

---

### 13. `find`: search for files

`find` walks a directory tree and lists files that match conditions: name, type, size, age, permissions, owner. It can also run a command on each result.

```bash
find <where> <conditions> <action>
find . -name "*.js"
```

**Common tests**

| Test | Meaning |
|---|---|
| `-name "pattern"` | Name matches a glob (case-sensitive). **Quote the pattern** |
| `-iname "pattern"` | Same, but case-insensitive |
| `-type f` / `-type d` / `-type l` | Files / directories / symlinks only |
| `-size +100M` / `-size -1k` | Bigger than 100 MB / smaller than 1 KB (`k`, `M`, `G`) |
| `-mtime -7` / `-mtime +30` | Modified less than 7 days ago / more than 30 days ago |
| `-mmin -60` | Modified in the last 60 minutes |
| `-empty` | Empty files or directories |
| `-user harsh` | Owned by a user |
| `-perm 777` | Has exactly these permissions |
| `-maxdepth N` | Don't go more than N levels deep |
| `-path "*/node_modules" -prune` | Skip a directory entirely |
| `-not` / `!` | Negate a test |
| `-o` | OR between tests (AND is the default) |

**Actions**

| Action | Meaning |
|---|---|
| `-print` | Print the path (the default) |
| `-delete` | Delete each match. Careful |
| `-exec cmd {} \;` | Run `cmd` once per file (`{}` is replaced by the path) |
| `-exec cmd {} +` | Run `cmd` once with all the files as arguments (much faster) |

**Examples**

```bash
find . -name "*.log"                          # all .log files under here
find . -iname "readme*"                       # README.md, readme.txt...
find . -type d -name node_modules             # every node_modules folder
find /var/log -type f -size +100M             # huge log files
find . -mtime -1                              # changed in the last 24 hours
find . -maxdepth 1 -type f                    # files in this folder only, no subfolders
find . -type f -empty                         # empty files
find . -name "*.js" -not -path "*/node_modules/*"   # skip node_modules
find . \( -name "*.jpg" -o -name "*.png" \)   # jpg OR png
```

**Doing things with the results**

```bash
find . -name "*.tmp" -delete                          # delete all .tmp files
find . -type d -name node_modules -prune -exec rm -rf {} +   # remove every node_modules
find . -name "*.sh" -exec chmod +x {} +               # make all scripts executable
find . -name "*.js" -exec grep -l "fetch(" {} +       # which JS files call fetch?
find /backups -mtime +30 -delete                      # delete backups older than 30 days
```

**`grep` vs `find`**

- **`grep`** searches **inside** files for text.
- **`find`** searches **for** files by their properties.
- Together: `find` picks the files and `grep` searches inside them.

**Gotchas**

- Always quote globs: `find . -name "*.js"`. Unquoted, the shell may expand `*.js` into the files in the current folder first, and you get wrong results or an error.
- Before `-delete`, run the same command **without** `-delete` to see what would go.
- `-delete` must come **last**. `find . -delete -name "*.tmp"` deletes **everything**, because actions run in order.
- `locate filename` is a faster alternative that searches a prebuilt index (update it with `sudo updatedb`), but its results can be out of date.

---

## 5. Permissions

Every file and directory has an **owner**, a **group**, and three sets of permissions: for the **u**ser (owner), the **g**roup, and **o**thers (everyone else).

```
-rwxr-xr--   harsh  devs   deploy.sh
 └┬┘└┬┘└┬┘
  u  g  o

r = read (4)   w = write (2)   x = execute (1)   - = not allowed
```

What each permission means depends on whether it's on a file or a directory:

| | On a **file** | On a **directory** |
|---|---|---|
| `r` | Read its contents | List the names inside (`ls`) |
| `w` | Change its contents | Create, delete or rename files inside |
| `x` | Run it as a program | Enter it (`cd`) and reach files inside |

### 14. `chmod`: change permissions

**Ch**ange **mod**e sets who can read, write and execute a file. You can describe the permissions in two ways.

**Numeric (octal) mode**: one digit each for user, group and others. Each digit is the sum of r (4), w (2) and x (1).

```
7 = 4+2+1 = rwx      6 = 4+2 = rw-      5 = 4+1 = r-x
4 = r--              0 = ---
```

```bash
chmod 755 deploy.sh       # rwxr-xr-x
chmod 644 notes.md        # rw-r--r--
chmod 600 ~/.ssh/id_rsa   # rw------- (SSH refuses keys that are too open)
chmod 700 ~/private/      # only you can enter
```

**Symbolic mode**: who (`u`, `g`, `o`, `a` = all), then an operator (`+` add, `-` remove, `=` set exactly), then which permissions.

```bash
chmod +x script.sh        # add execute for everyone
chmod u+x script.sh       # add execute for the owner only
chmod g+w shared.txt      # let the group write
chmod o-r secret.txt      # take read away from others
chmod go-rwx private.key  # remove everything for group and others
chmod u=rw,go=r file.txt  # set exactly: same as 644
```

**Common values**

| Number | Symbolic | Typical use |
|---|---|---|
| `755` | `rwxr-xr-x` | Scripts, programs, directories |
| `644` | `rw-r--r--` | Normal files (code, configs, HTML) |
| `700` | `rwx------` | Private directories |
| `600` | `rw-------` | Secrets: SSH keys, `.env`, credentials |
| `777` | `rwxrwxrwx` | Everyone can do everything. **Almost never correct** |

**Options**

| Flag | Meaning |
|---|---|
| `-R` | Apply recursively to everything inside a directory |
| `-v` | Print every file processed |
| `-c` | Print only the files that actually changed |

**Gotchas**

- `Permission denied` when running `./script.sh` usually means it needs `chmod +x`.
- `chmod -R 755` also makes every *file* executable. To set directories and files differently:

```bash
find . -type d -exec chmod 755 {} +
find . -type f -exec chmod 644 {} +
```

- "Just `chmod 777` it" hides the real problem (usually the wrong owner, which `chown` fixes) and opens a security hole.

---

### 15. `chown`: change owner

**Ch**ange **own**er sets which user, and optionally which group, owns a file. Changing ownership to someone else normally needs `sudo`.

```bash
sudo chown harsh file.txt             # change the owner
sudo chown harsh:devs file.txt        # change owner and group
sudo chown :devs file.txt             # change only the group (same as chgrp devs)
sudo chown -R www-data:www-data /var/www    # a whole web root
sudo chown -R $USER:$USER ~/project   # take back files that were created as root
```

**Options**

| Flag | Meaning |
|---|---|
| `-R` | Recursive |
| `-v` / `-c` | Print every file / only the files that changed |
| `--reference=file` | Copy the owner and group from another file |
| `-h` | Change the symlink itself, not the file it points to |

**When you'll use it**

- You ran something with `sudo` (e.g. `sudo npm install`) and now your own files belong to root, so you get `EACCES` or `Permission denied`. Fix: `sudo chown -R $USER:$USER .`
- Web servers (nginx, Apache) run as a user like `www-data` and need to own, or at least be able to read, the files they serve.
- Docker volume mounts create files owned by root or by some numeric user ID.

**Related commands**

```bash
whoami              # your username
id                  # your user ID, group ID, and every group you're in
groups              # just your groups
ls -l file          # see the current owner and group
```

---

### 16. `sudo`: run as root

`sudo` ("**s**uper**u**ser **do**") runs a single command as root, the administrator account that can do anything. It asks for **your** password (not root's), and only works if your user is allowed to use it (usually by being in the `sudo` or `wheel` group).

```bash
sudo apt update                       # install and update packages
sudo systemctl restart nginx          # manage services
sudo nano /etc/hosts                  # edit system files
sudo !!                               # re-run your last command with sudo
```

**Common options**

| Flag | Meaning |
|---|---|
| `-i` | Start a login shell as root (you *become* root until you `exit`) |
| `-s` | Start a shell as root but keep your current environment |
| `-u user` | Run as another user instead of root: `sudo -u postgres psql` |
| `-l` | List what you're allowed to run with sudo |
| `-k` | Forget your cached password now (the next sudo asks again) |
| `-E` | Keep your environment variables |

**How it works**

- After you enter your password, sudo remembers it for about 15 minutes, so the next few `sudo` commands don't ask.
- Who may use sudo is set in `/etc/sudoers`. **Edit it only with `sudo visudo`**, which checks the syntax. A broken sudoers file can lock you out of sudo entirely.
- Every sudo command is logged (in `/var/log/auth.log` or `journalctl`).

**Gotchas**

- Redirects are handled by *your* shell, not by sudo, so `sudo echo "x" > /etc/file` still fails with permission denied. Use `tee`:

```bash
echo "127.0.0.1 myapp.local" | sudo tee -a /etc/hosts
```

- Don't use sudo to get around permission errors in your own projects (`sudo npm install`, `sudo pip install`). It leaves root-owned files behind and causes more errors later. Fix ownership with `chown` instead.
- Be extra careful with any command that has both `sudo` and `rm`.

---

## 6. Processes

A **process** is a running program. Each process has a **PID** (process ID), an owner, a parent process, and uses some amount of CPU and memory.

### 17. `ps` and `top`: what's running?

**`ps`** (**p**rocess **s**tatus) takes a **snapshot** of the processes running right now.

```bash
ps                        # just the processes in this terminal
ps aux                    # every process on the system (BSD style, the most common)
ps -ef                    # every process (Unix style: shows the parent PID)
ps aux | grep node        # find a specific process
ps aux --sort=-%mem | head    # the biggest memory users
ps aux --sort=-%cpu | head    # the biggest CPU users
ps -ef --forest           # show parent/child relationships as a tree
```

`aux` means: **a**ll users, show the **u**ser, and include processes with no terminal (**x**).

**Reading `ps aux` output**

```
USER   PID  %CPU %MEM    VSZ   RSS TTY  STAT START  TIME COMMAND
harsh  4321  2.1  1.5 812345 61234 ?    Sl   10:02  0:14 node server.js
```

| Column | Meaning |
|---|---|
| `USER` | Who owns the process |
| `PID` | Process ID: what you pass to `kill` |
| `%CPU` / `%MEM` | Share of CPU / RAM it's using |
| `RSS` | Actual memory in use (KB) |
| `STAT` | State: `R` running, `S` sleeping, `D` waiting on disk, `Z` zombie, `T` stopped |
| `START` / `TIME` | When it started / total CPU time used |
| `COMMAND` | The command that started it |

**`top`** shows a **live**, updating view, like a task manager in the terminal.

```bash
top
top -u harsh              # only your processes
top -p 4321               # watch a single PID
```

| Key in `top` | Action |
|---|---|
| `P` | Sort by CPU |
| `M` | Sort by memory |
| `k` | Kill a process (asks for the PID) |
| `1` | Show each CPU core separately |
| `c` | Show full command lines |
| `u` | Filter by user |
| `q` | Quit |

The header shows **load average** (three numbers: the last 1, 5 and 15 minutes). A rough rule: if the load is consistently above your number of CPU cores (`nproc`), the machine is overloaded.

**Better alternatives**

- `htop`: a colorful, scrollable `top` that you can use with the mouse (`sudo apt install htop`).
- `pgrep -a node`: find PIDs by name, without the `grep` matching itself.
- `pstree`: the process tree.
- `free -h`: a quick summary of memory use.
- `lsof -i :3000` or `ss -ltnp`: which process is using a port.

---

### 18. `kill`: stop a process

Despite its name, `kill` sends a **signal** to a process. Most signals ask the process to stop, but some do other things.

```bash
kill 4321                 # send SIGTERM (15): "please shut down"
kill -9 4321              # send SIGKILL (9): stop right now, can't be ignored
kill -HUP 4321            # send SIGHUP: many daemons reload their config on this
kill -l                   # list every signal
```

**The signals that matter**

| Signal | Number | What it means | Can the process catch it? |
|---|---|---|---|
| `SIGTERM` | 15 | Polite request to shut down (the default) | Yes: it can save state and close connections |
| `SIGINT` | 2 | Interrupt: what `Ctrl+C` sends | Yes |
| `SIGKILL` | 9 | The kernel stops it immediately | **No** |
| `SIGHUP` | 1 | "Hang up": often used to mean "reload config" | Yes |
| `SIGSTOP` / `SIGCONT` | 19 / 18 | Pause / resume (`Ctrl+Z` sends a similar signal, `SIGTSTP`) | STOP: no |

**Killing by name**

```bash
pkill node                # kill every process whose name matches "node"
pkill -f "server.js"      # match against the full command line
killall nginx             # kill every process named exactly "nginx"
pgrep -a node             # preview what pkill would hit
```

**Typical workflow: "port 3000 is already in use"**

```bash
lsof -i :3000             # find the PID using the port
kill 4321                 # ask it to stop
kill -9 4321              # only if it's still there after a few seconds
# one-liner:
kill $(lsof -t -i :3000)
```

**Gotchas**

- **Try plain `kill` (SIGTERM) first.** `kill -9` gives the process no chance to clean up, which can leave corrupt files, stale lock files, or half-written data.
- You can only kill your own processes. Other users' and system processes need `sudo`.
- **Zombie** processes (`Z` in `ps`) are already dead and can't be killed. Their parent process needs to clean them up, or be killed itself.
- Related job-control commands: `Ctrl+Z` pauses the current command, `bg` resumes it in the background, `fg` brings it back, `jobs` lists them, and `command &` starts something in the background.

---

## 7. System and network

### 19. `df` and `du`: disk space

These two answer different questions:

- **`df`** (**d**isk **f**ree): how full is each **disk or partition**?
- **`du`** (**d**isk **u**sage): how much space do these **files and folders** take?

**`df`**

```bash
df -h                     # every mounted filesystem, readable sizes
df -h /                   # just the one holding /
df -h .                   # just the one holding the current directory
df -i                     # inodes (file count) instead of bytes
df -hT                    # also show the filesystem type (ext4, xfs, tmpfs...)
```

```
Filesystem      Size  Used Avail Use% Mounted on
/dev/nvme0n1p2  468G  301G  144G  68% /
tmpfs           7.8G  2.1M  7.8G   1% /dev/shm
```

**`du`**

| Flag | Meaning |
|---|---|
| `-h` | Human-readable sizes |
| `-s` | Summary: one total per argument instead of every subfolder |
| `-c` | Add a grand total at the end |
| `-d N` / `--max-depth=N` | Only show N levels deep |
| `-a` | Include files, not just directories |
| `--exclude=pattern` | Skip matching paths |

```bash
du -sh folder/                    # total size of one folder
du -sh *                          # size of each item in the current directory
du -sh * | sort -h                # the same, sorted smallest to largest
du -h -d 1 ~ | sort -rh | head    # biggest folders in your home directory
du -sh node_modules               # the classic shock
du -sh .[!.]* *                   # include hidden folders like .cache and .git
```

**Typical workflow: "disk full"**

```bash
df -h                                   # 1. which disk is full?
sudo du -h -d 1 / 2>/dev/null | sort -rh | head   # 2. which top-level folder is big?
sudo du -h -d 1 /var | sort -rh | head  # 3. drill down (often /var/log or /var/lib/docker)
```

Common culprits: old logs, Docker images (`docker system df`, then `docker system prune`), `node_modules`, package caches (`sudo apt clean`), `~/.cache`.

**Gotchas**

- `df` can show a full disk while `du` finds nothing big. That usually means a **deleted file is still open** by some process, which keeps its space in use. Find it with `sudo lsof +L1`, then restart that process.
- A disk can "fill up" on **inodes** (too many tiny files) even when there's free space. `df -i` shows this.
- `ncdu` is an interactive `du` that lets you browse and delete, which is much nicer for cleanups.

---

### 20. `curl`: talk to the web

`curl` ("**c**lient **URL**") sends requests to URLs and prints the response. It speaks HTTP, HTTPS, FTP and more. For developers it works as a terminal Postman: testing APIs, checking whether a service is up, and downloading files.

```bash
curl https://api.github.com               # a simple GET; prints the response body
```

**Common options**

| Flag | Meaning |
|---|---|
| `-X METHOD` | HTTP method: `GET`, `POST`, `PUT`, `PATCH`, `DELETE` |
| `-H "Key: Value"` | Add a header |
| `-d 'data'` | Send a request body (turns the request into a POST) |
| `--json 'data'` | Send JSON and set the JSON headers for you (curl 7.82+) |
| `-i` | Include the response headers in the output |
| `-I` | Headers only (a HEAD request) |
| `-v` | Verbose: show the whole conversation, including TLS and request headers |
| `-s` | Silent: hide the progress bar |
| `-o file` | Save the output to a file |
| `-O` | Save using the file name from the URL |
| `-L` | Follow redirects (301/302) |
| `-u user:pass` | Basic auth |
| `-k` | Skip TLS certificate checks (**only** for local testing) |
| `-w "..."` | Print extra information, e.g. the status code or timing |
| `-F` | Upload a form or file (multipart) |
| `--max-time N` | Give up after N seconds |

**API examples**

```bash
# GET with an auth header
curl -H "Authorization: Bearer $TOKEN" https://api.example.com/me

# POST JSON
curl -X POST http://localhost:3000/users \
     -H "Content-Type: application/json" \
     -d '{"name":"harsh","role":"admin"}'

# the same, shorter (newer curl)
curl --json '{"name":"harsh"}' http://localhost:3000/users

# PUT and DELETE
curl -X PUT -d '{"name":"new"}' -H "Content-Type: application/json" http://localhost:3000/users/1
curl -X DELETE http://localhost:3000/users/1

# upload a file
curl -F "avatar=@photo.jpg" http://localhost:3000/upload

# pretty-print JSON responses (needs jq)
curl -s https://api.github.com/users/octocat | jq .
```

**Debugging examples**

```bash
curl -I https://example.com                         # status code and headers only
curl -sL -o /dev/null -w "%{http_code}\n" https://example.com   # just the status code
curl -v https://example.com                         # see the TLS handshake and headers
curl -w "total: %{time_total}s\n" -o /dev/null -s https://example.com   # how long did it take?
curl http://localhost:8080/health                   # is my service up?
```

**Downloading**

```bash
curl -O https://example.com/file.zip                # keep the original name
curl -L -o node.tar.gz https://nodejs.org/dist/...  # follow redirects, choose the name
curl -C - -O https://example.com/big.iso            # resume an interrupted download
```

**Gotchas**

- Without `-L`, curl doesn't follow redirects, so you may just get an empty `301` response.
- `-d` sends `Content-Type: application/x-www-form-urlencoded` by default. For a JSON API, add `-H "Content-Type: application/json"` or use `--json`.
- Use single quotes around JSON so the shell leaves `"` and `$` alone.
- Tokens you type on the command line end up in your shell history. Put them in an environment variable (`$TOKEN`) instead.
- Never pipe a script from the internet straight into a shell (`curl ... | bash`) unless you trust the source. Download it and read it first.
- `wget` is an alternative that's simpler for downloads (`wget URL`) and can download whole sites recursively.

---

## Bonus: glue that makes it all work

These aren't commands, but they let you combine the 20 above.

| Syntax | Meaning | Example |
|---|---|---|
| `\|` | Pipe: send one command's output into another | `ps aux \| grep node` |
| `>` | Write output to a file (overwrite) | `ls > files.txt` |
| `>>` | Append output to a file | `echo "done" >> log.txt` |
| `2>` | Redirect errors | `find / -name x 2>/dev/null` |
| `&>` | Redirect output and errors | `npm run build &> build.log` |
| `&&` | Run the next command only if the previous one succeeded | `npm install && npm start` |
| `\|\|` | Run the next command only if the previous one failed | `ping -c1 host \|\| echo "down"` |
| `;` | Run the next command either way | `cd /tmp; ls` |
| `&` | Run in the background | `node server.js &` |
| `$(...)` | Insert a command's output | `echo "Today is $(date)"` |
| `*` | Wildcard: match anything | `rm *.log` |
| `Tab` | Autocomplete file and command names | |
| `Ctrl+C` | Stop the running command | |
| `Ctrl+R` | Search your command history | |
| `!!` | The previous command | `sudo !!` |
| `history` | Show previous commands | `history \| grep ssh` |

### Putting it together

```bash
# Top 5 IPs hitting your server
cat access.log | awk '{print $1}' | sort | uniq -c | sort -rn | head -5

# Find the 10 biggest files under the current directory
find . -type f -exec du -h {} + | sort -rh | head -10

# Watch a log for errors only
tail -f app.log | grep -i error

# Count lines of JavaScript in a project, skipping node_modules
find . -name "*.js" -not -path "*/node_modules/*" -exec cat {} + | wc -l

# Kill whatever is on port 3000, then start the app
kill $(lsof -t -i :3000) 2>/dev/null; npm start
```

---

## Summary

- **Navigate:** `pwd` (where am I), `ls` (what's here), `cd` (go there)
- **Manage files:** `mkdir -p`, `touch`, `cp -r`, `mv` (also renames), `rm` (permanent, so be careful)
- **Read files:** `cat` (small files), `less` (big files), `head` / `tail -f` (the start, the end, live logs)
- **Search:** `grep` (text inside files), `find` (files by name, size, age)
- **Permissions:** `chmod` (755 / 644 / 600), `chown` (fix ownership), `sudo` (root, only when needed)
- **Processes:** `ps aux` / `top` (what's running), `kill` (SIGTERM first, `-9` last)
- **System and network:** `df -h` (disk free), `du -sh` (folder sizes), `curl` (HTTP from the terminal)
- **When stuck:** `man <command>` or `<command> --help`
