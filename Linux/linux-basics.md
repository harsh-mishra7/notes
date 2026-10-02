# Linux Basics

## Brief

Linux is the operating system that runs most of the world's servers, cloud machines, Docker containers, Android phones, and CI runners. If you deploy code anywhere, it almost certainly ends up on Linux.

This note covers the ideas you need before the commands make sense: what Linux actually is, how the filesystem is laid out, what the shell does, how users and permissions work, what processes are, and how to install software. Each section ends with the handful of commands that go with it.

---

## The analogy that makes it click

**Linux is a building.**

```
┌──────────────────────────────────────────────┐
│  You (typing commands)                       │
├──────────────────────────────────────────────┤
│  Shell (bash, zsh)     ← the receptionist    │
├──────────────────────────────────────────────┤
│  Programs (ls, git, node, nginx)             │
├──────────────────────────────────────────────┤
│  Kernel                ← the building manager│
├──────────────────────────────────────────────┤
│  Hardware (CPU, RAM, disk, network)          │
└──────────────────────────────────────────────┘
```

- The **hardware** is the physical building: rooms, power, plumbing.
- The **kernel** is the building manager. It decides who gets which room (memory), who gets the elevator next (CPU), and who can open which door (permissions). Programs never touch the hardware directly; they ask the kernel.
- **Programs** are the tenants doing actual work.
- The **shell** is the receptionist. You tell it what you want in words, and it finds the right tenant to do it.

---

## 1. What "Linux" actually means

Strictly speaking, **Linux is only the kernel**, written by Linus Torvalds in 1991. What people call "Linux" is the kernel plus a pile of tools (shell, core utilities, package manager, init system) bundled together as a **distribution** ("distro").

| Distro family | Examples | Package manager | Where you see it |
|---|---|---|---|
| Debian | Debian, Ubuntu, Mint | `apt` (`.deb`) | Most cloud VMs, desktops |
| Red Hat | RHEL, Fedora, Rocky, Amazon Linux | `dnf` / `yum` (`.rpm`) | Enterprise servers, AWS |
| Alpine | Alpine | `apk` | Tiny Docker images |
| Arch | Arch, Manjaro | `pacman` | Enthusiast desktops |

The commands in this note work the same on all of them. Mostly what changes between distros is how you install software.

```bash
uname -a              # kernel version and architecture
cat /etc/os-release   # which distro and version
```

---

## 2. Everything is a file

This is the most important Linux idea. Regular files, directories, hard disks, your terminal, running processes, and even random-number generators all show up as **files** in one tree.

```bash
cat /proc/cpuinfo      # CPU info, generated live by the kernel
cat /proc/meminfo      # memory info
ls /dev                # devices: disks (sda, nvme0n1), terminals (tty), etc.
echo hi > /dev/null    # the "black hole": anything written here disappears
head -c 16 /dev/urandom | xxd   # random bytes
```

Because everything is a file, the same small set of tools (`cat`, `grep`, `>`, `|`) works on all of it.

---

## 3. The filesystem tree

Windows has drive letters (`C:\`, `D:\`). Linux has **one tree** starting at `/` (called "root"). Extra disks and USB drives get **mounted** somewhere inside that tree.

```
/
├── bin, usr/bin   → programs everyone can run (ls, cp, git)
├── sbin           → admin programs (fdisk, reboot)
├── etc            → configuration files (nginx.conf, hosts, passwd)
├── home           → personal folders: /home/harsh, /home/alice
├── root           → home folder of the root (admin) user
├── var            → data that changes: logs (/var/log), caches, databases
├── tmp            → temporary files, wiped on reboot
├── opt            → optional / third-party software
├── dev            → device files
├── proc, sys      → live kernel and process info (not on disk)
└── mnt, media     → mount points for extra disks and USB drives
```

Rules of thumb:
- Config broken? Look in `/etc`.
- App misbehaving? Look in `/var/log`.
- Your own stuff lives in `/home/<you>`, which you can write as `~`.

### Paths

| Path | Meaning |
|---|---|
| `/home/harsh/notes` | **Absolute** path: starts at root, works from anywhere |
| `notes/linux.md` | **Relative** path: starts from where you are now |
| `.` | The current directory |
| `..` | The parent directory |
| `~` | Your home directory |
| `-` | (with `cd`) the previous directory |

Linux is **case-sensitive**: `Notes`, `notes`, and `NOTES` are three different files. Files starting with `.` (like `.bashrc`, `.git`) are **hidden** and only show with `ls -a`.

```bash
pwd                 # where am I?
ls -la              # list everything here, including hidden files
cd /var/log         # absolute
cd ../..            # up two levels
cd ~                # home (plain `cd` does the same)
cd -                # back to where I just was
```

---

## 4. The shell and the terminal

- The **terminal** is the window you type into.
- The **shell** is the program inside it that reads your command, runs it, and prints the result. The common ones are `bash` (the default on most servers) and `zsh` (the default on macOS).

```bash
echo $SHELL        # which shell am I using?
```

### Anatomy of a command

```
command   -flags          arguments
  ls       -lah            /var/log
  verb     how             to what
```

- Short flags use one dash and can be combined: `-l -a -h` = `-lah`.
- Long flags use two dashes: `--all`, `--help`.

### Getting help

```bash
man ls             # full manual (q to quit, /word to search)
ls --help          # short summary
type cd            # is it a builtin, alias, or program?
which python3      # where is this program installed?
```

### Shortcuts worth learning on day one

| Key | Does |
|---|---|
| `Tab` | Autocomplete file and command names (press twice to see options) |
| `↑` / `↓` | Scroll through previous commands |
| `Ctrl + R` | Search command history |
| `Ctrl + C` | Stop the running command |
| `Ctrl + D` | End input / exit the shell |
| `Ctrl + L` | Clear the screen |
| `Ctrl + A` / `Ctrl + E` | Jump to start / end of the line |
| `!!` | Repeat the last command (`sudo !!` is a classic) |

---

## 5. Working with files and directories

```bash
mkdir projects                 # make a directory
mkdir -p a/b/c                 # make nested directories in one go
touch notes.txt                # create an empty file (or update its timestamp)

cp notes.txt backup.txt        # copy a file
cp -r projects projects-bak    # copy a directory (-r = recursive)
mv backup.txt old.txt          # rename
mv old.txt ~/archive/          # move

rm old.txt                     # delete a file
rm -r projects-bak             # delete a directory and everything in it
rmdir empty-dir                # delete an empty directory
```

> ⚠️ There is **no recycle bin**. `rm` deletes immediately. Be careful with `rm -rf`, and never run it on a path built from a variable you haven't checked.

### Reading files

```bash
cat file.txt          # print the whole file
less big.log          # scroll through it (q to quit, / to search)
head -n 20 file.txt   # first 20 lines
tail -n 20 file.txt   # last 20 lines
tail -f app.log       # keep printing new lines as they're written (logs!)
wc -l file.txt        # count lines
```

### Editing files

You will eventually need to edit a file on a server with no GUI.

- **nano**: easy. `nano file.txt`, edit, `Ctrl + O` to save, `Ctrl + X` to exit.
- **vim**: powerful, confusing at first. `vim file.txt`, press `i` to type, `Esc` to stop typing, `:wq` to save and quit, `:q!` to quit without saving.

### Wildcards (globbing)

The shell expands these *before* the command runs:

```bash
ls *.md          # every file ending in .md
ls report?.pdf   # report1.pdf, reportA.pdf (? = exactly one character)
ls img[0-9].png  # img0.png ... img9.png
cp *.{jpg,png} pics/   # brace expansion: *.jpg and *.png
```

---

## 6. Users, groups, and permissions

Linux was built for many people sharing one machine, so **every file has an owner and permissions**.

- Every person (and many services, like `www-data` or `postgres`) is a **user**.
- Users belong to **groups** (e.g. `docker`, `sudo`).
- **root** is the superuser: it can do anything, including destroy the system.

```bash
whoami           # who am I?
id               # my user id, group id, and groups
groups           # just the groups
```

### Reading permissions

```bash
$ ls -l deploy.sh
-rwxr-xr-- 1 harsh devs 512 Oct  1 10:00 deploy.sh
```

```
 -    rwx    r-x    r--     harsh   devs
type  owner  group  others  owner   group
```

| Letter | On a file | On a directory |
|---|---|---|
| `r` (4) | read contents | list what's inside |
| `w` (2) | change contents | create / delete files inside |
| `x` (1) | run it as a program | `cd` into it |

The first character is the type: `-` file, `d` directory, `l` symlink.

Permissions are often written as numbers, adding up each group of three: `rwx` = 4+2+1 = **7**, `r-x` = **5**, `r--` = **4**. So `rwxr-xr--` = **754**.

```bash
chmod +x deploy.sh         # make it executable
chmod 644 config.yml       # rw-r--r--  (typical file)
chmod 755 bin/             # rwxr-xr-x  (typical directory / script)
chmod 600 ~/.ssh/id_rsa    # rw-------  (secrets: only me)
chown harsh:devs file.txt  # change owner and group
```

### sudo

You normally work as a regular user. When you need admin power for one command, use `sudo` ("superuser do"):

```bash
sudo apt update
sudo systemctl restart nginx
```

It asks for **your** password, and only works if you're in the `sudo` (Ubuntu) or `wheel` (Red Hat) group. Working as a normal user and reaching for `sudo` only when needed limits how much damage a typo can do.

---

## 7. Processes

Every running program is a **process** with a unique **PID** (process ID). Processes have a parent (the shell that started them, for example), an owner, and a state.

```bash
ps aux                  # every process on the system
ps aux | grep node      # find a specific one
top                     # live view of CPU and memory (q to quit)
htop                    # nicer top, if installed
```

### Stopping processes

```bash
kill 1234        # politely ask PID 1234 to stop (SIGTERM)
kill -9 1234     # force kill (SIGKILL), only if the polite way fails
pkill node       # kill by name
```

### Foreground and background

```bash
npm run dev        # runs in the foreground; the terminal is busy
# Ctrl + Z         → pause it
bg                 # resume it in the background
fg                 # bring it back to the foreground
jobs               # list background jobs

long-task &              # start directly in the background
nohup long-task &        # ...and keep it running after you log out
```

### Services (systemd)

Long-running background programs like web servers and databases are **services** (also called daemons, which is where names like `sshd` and `dockerd` come from). On most modern distros, **systemd** manages them:

```bash
systemctl status nginx       # is it running? recent logs
sudo systemctl start nginx
sudo systemctl stop nginx
sudo systemctl restart nginx
sudo systemctl enable nginx  # start automatically on boot
journalctl -u nginx -f       # follow a service's logs
```

---

## 8. Installing software (package managers)

You rarely download installers on Linux. A **package manager** downloads software from trusted repositories, installs it, pulls in its dependencies, and keeps it updated.

| Task | Debian / Ubuntu | Fedora / RHEL | Alpine |
|---|---|---|---|
| Refresh package list | `sudo apt update` | (automatic) | `apk update` |
| Install | `sudo apt install git` | `sudo dnf install git` | `apk add git` |
| Remove | `sudo apt remove git` | `sudo dnf remove git` | `apk del git` |
| Upgrade everything | `sudo apt upgrade` | `sudo dnf upgrade` | `apk upgrade` |
| Search | `apt search nginx` | `dnf search nginx` | `apk search nginx` |

---

## 9. Input, output, pipes, and redirection

This is where the shell becomes really useful. Every process has three standard streams:

| Stream | Number | Default |
|---|---|---|
| **stdin** (input) | 0 | keyboard |
| **stdout** (output) | 1 | screen |
| **stderr** (errors) | 2 | screen |

### Redirection: send output to a file

```bash
ls > files.txt          # write stdout to a file (overwrites)
ls >> files.txt         # append instead
cmd 2> errors.txt       # write only errors to a file
cmd > out.txt 2>&1      # both output and errors to the same file
cmd > /dev/null 2>&1    # throw everything away
sort < names.txt        # read stdin from a file
```

### Pipes: send output to another command

The `|` takes the output of one command and feeds it as input to the next. That's the Unix philosophy: **small tools that each do one thing, chained together.**

```bash
cat access.log | grep 500 | wc -l         # how many 500 errors?
ps aux | grep python                      # find python processes
history | grep docker                     # which docker commands did I run?
du -sh * | sort -h | tail -5              # 5 biggest things here
```

### Chaining commands

```bash
mkdir build && cd build    # run the second only if the first succeeds
make || echo "failed"      # run the second only if the first fails
cmd1 ; cmd2                # run both, no matter what
```

---

## 10. Searching

```bash
grep "error" app.log            # lines containing "error"
grep -i "error" app.log         # case-insensitive
grep -rn "TODO" src/            # search a whole directory, show line numbers

find . -name "*.log"            # files by name, under the current directory
find / -size +100M 2>/dev/null  # files bigger than 100 MB
find . -mtime -1                # modified in the last day
```

---

## 11. Environment variables and PATH

**Environment variables** are key-value settings that every program inherits from the shell.

```bash
echo $HOME           # /home/harsh
echo $USER           # harsh
env                  # list all of them

export API_KEY=abc123        # set for this session (and child processes)
API_KEY=abc123 node app.js   # set for one command only
```

### PATH

When you type `git`, how does the shell find it? It checks every directory listed in `$PATH`, in order:

```bash
$ echo $PATH
/usr/local/bin:/usr/bin:/bin:/home/harsh/.local/bin
```

If you get `command not found` for something you just installed, its directory is probably missing from `PATH`. To run a script in the current directory, which is *not* on PATH, use `./script.sh`.

### Making settings permanent

Variables you `export` disappear when you close the terminal. To keep them, add the line to your shell's startup file: `~/.bashrc` for bash, `~/.zshrc` for zsh. Then reload:

```bash
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

The same file is where **aliases** go:

```bash
alias ll='ls -lah'
alias gs='git status'
```

---

## 12. Disk, memory, and system info

```bash
df -h          # free space on each disk
du -sh *       # size of each item in the current directory
free -h        # RAM and swap usage
uptime         # how long it's been up, plus load average
nproc          # number of CPU cores
lsblk          # list disks and partitions
```

---

## 13. Networking basics

```bash
ip a                        # my IP addresses (older: ifconfig)
ping google.com             # can I reach it? (Ctrl + C to stop)
curl https://api.example.com/health   # make an HTTP request
curl -I https://example.com           # headers only
wget https://example.com/file.zip     # download a file
ss -tulpn                   # which ports are open and which process owns them
dig example.com             # DNS lookup
ssh user@server-ip          # log into a remote machine
scp file.txt user@server:/tmp/        # copy a file to a remote machine
```

---

## 14. Archives and compression

```bash
tar -czf backup.tar.gz folder/     # create   (c = create, z = gzip, f = file)
tar -xzf backup.tar.gz             # extract  (x = extract)
tar -tzf backup.tar.gz             # list contents without extracting
zip -r site.zip site/
unzip site.zip
```

A mnemonic for `tar`: **c**reate **z**e **f**ile and e**x**tract **z**e **f**ile.

---

## Cheat sheet

| I want to... | Command |
|---|---|
| See where I am | `pwd` |
| List files (incl. hidden) | `ls -la` |
| Move around | `cd path`, `cd ..`, `cd ~`, `cd -` |
| Create a directory / file | `mkdir -p dir`, `touch file` |
| Copy / move / delete | `cp -r`, `mv`, `rm -r` |
| Read a file | `cat`, `less`, `head`, `tail -f` |
| Edit a file | `nano file`, `vim file` |
| Search text | `grep -rn "text" dir/` |
| Find files | `find . -name "*.js"` |
| Change permissions | `chmod 755 file`, `chmod +x file` |
| Change owner | `chown user:group file` |
| Run as admin | `sudo cmd` |
| See processes | `ps aux`, `top`, `htop` |
| Kill a process | `kill PID`, `kill -9 PID`, `pkill name` |
| Manage a service | `systemctl status/start/stop/restart name` |
| Install software | `sudo apt install pkg` |
| Check disk / memory | `df -h`, `du -sh *`, `free -h` |
| Make a request | `curl url` |
| Log into a server | `ssh user@host` |
| Get help | `man cmd`, `cmd --help` |

---

## Summary

- **Linux** is a kernel; a **distro** is the kernel plus tools and a package manager.
- **Everything is a file**, arranged in one tree starting at `/`. Config is in `/etc`, logs are in `/var/log`, your files are in `~`.
- The **shell** reads `command -flags arguments`. Use `Tab`, `Ctrl + R`, and `man` all the time.
- Every file has an **owner, a group, and `rwx` permissions**. Use `sudo` only when you need it.
- Every running program is a **process** with a PID. Long-running ones are **services** managed by `systemctl`.
- Install software with the **package manager** (`apt`, `dnf`, `apk`).
- **Pipes (`|`) and redirection (`>`, `>>`, `2>&1`)** let you combine small tools into powerful one-liners.
- **Environment variables** and `PATH` decide how programs are configured and found. Make them permanent in `~/.bashrc`.
