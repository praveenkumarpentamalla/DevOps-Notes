# The Complete Bash Scripting Course
### Beginner → Advanced → Expert, for DevOps / Platform Engineering / Cloud Automation

**Legend:** ⭐ MUST KNOW · 🔥 ADVANCED · 🚀 EXPERT · 🏭 PRODUCTION CRITICAL

**How this course is scoped for you:** You already know Linux, Docker, Kubernetes, Terraform, AWS, Git, GitHub Actions, Nginx, PostgreSQL, and networking. Linux admin basics are only explained when they change *how Bash behaves* (e.g., process substitution, file descriptors, signals). The weight of this course is on Bash-the-language, Bash-the-glue, and Bash-in-production.

**Learning path:**

```
Bash Fundamentals → Scripting → Automation → Advanced Bash
     → DevOps Automation → Production Engineering → Platform Engineering
```

**Master vs. Awareness for a Senior DevOps / Platform Engineer:**

| Master deeply (write blind, debug instantly) | Awareness only (know it exists, look up syntax) |
|---|---|
| Quoting, `set -euo pipefail`, traps, exit codes | POSIX `/bin/sh` portability minutiae |
| Arrays, parameter expansion, functions | `coproc`, namerefs (used rarely but powerfully) |
| `getopts`, argument parsing, CLI design | Obscure `printf` format specifiers |
| `grep`/`sed`/`awk`/`jq` for text & JSON | Full `awk` as a programming language (know basics; reach for Python beyond that) |
| Process mgmt, `trap`, signals, `flock` | GNU `parallel` internals |
| ShellCheck, security, idempotency, retries | Bash's internal parser/tokenizer |
| SSH/Docker/K8s/AWS/DB/API automation patterns | Deep systemd unit-file authoring (know enough to wire Bash into it) |

---

## PART 1 — Shell & Bash Fundamentals ⭐

### What is a shell, and what is Bash?

A **shell** is a program that reads commands you type (or that a script contains) and asks the kernel to execute them. It's the "operator console" between you and the OS. **Bash** (Bourne Again SHell) is one specific shell — a superset of the original Bourne shell (`sh`), with more features: arrays, `[[ ]]`, `${...}` expansions, process substitution, `select`, brace expansion, and more.

**Why this matters for scripting:** every "advanced" Bash feature you'll use (arrays, `[[`, `${var//x/y}`) is a Bash *extension*, not POSIX `sh`. If a script starts with `#!/bin/sh` on a system where `/bin/sh` is `dash` (Debian/Ubuntu) or `busybox sh` (Alpine), those features silently break or error. This single fact causes a huge share of "works on my machine" production bugs.

```bash
#!/bin/bash
echo "This uses bash-only syntax:"
arr=(a b c)          # Bash array — NOT valid in POSIX sh
echo "${arr[1]}"
```

### Bash vs sh vs Zsh

- **`sh`**: POSIX standard interface. On most Linux distros, `/bin/sh` is a symlink to `dash` (fast, minimal, POSIX-strict) — *not* Bash. macOS's `/bin/sh` is also not full Bash behavior-wise.
- **Bash**: the shell you write scripts in almost everywhere in DevOps (CI runners, containers, servers).
- **Zsh**: default interactive shell on modern macOS; great interactively, but production **scripts** should still target Bash for portability across Linux servers/containers, since that's what's installed everywhere by default.

**Rule of thumb 🏭:** if you write `#!/bin/bash` and use Bash features, the box must actually have `bash` installed (true for almost every Linux server, but *not* guaranteed in minimal containers like `alpine` — you need `apk add bash`).

### Shell execution model

When you run a script, a new process is created (`fork`), the interpreter named in the shebang is `exec`'d into that process, and it reads your script line by line, tokenizing → expanding → executing each command. Understanding "one new process per script run" explains why:
- variables set inside a script don't affect your parent shell (unless you `source` it)
- `cd` inside a script doesn't change your terminal's directory
- a `&` background job's lifetime is tied to that subshell

### Interactive vs non-interactive; login vs non-login

| | Interactive | Non-interactive |
|---|---|---|
| **Login** | You SSH in / open a terminal that logs you in | `bash -l script.sh`, some cron/systemd configs |
| **Non-login** | `bash` inside an already-open terminal | Running a script directly: `./script.sh`, CI runners |

This matters because **different startup files load in each case** (see below) — this is *the* #1 cause of "works when I run it by hand, fails under cron/systemd/CI."

### Shebang, permissions, running scripts ⭐

```bash
#!/bin/bash          # tells the kernel which interpreter to exec
```

The shebang is parsed by the kernel itself (via `execve`), not by your shell. `chmod +x script.sh` sets the execute bit so `./script.sh` works. Without execute permission, you must invoke it explicitly: `bash script.sh`.

```bash
chmod +x deploy.sh
./deploy.sh                  # uses the shebang interpreter
bash deploy.sh                # explicit — works even without +x
sh deploy.sh                  # forces POSIX sh semantics — dangerous if script uses bash-isms
```

### PATH and $PATH ⭐

`$PATH` is a colon-separated list of directories the shell searches, in order, to resolve a bare command name.

```bash
echo "$PATH"
# /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```

**Production trap 🏭:** cron and systemd run with a *much smaller* `$PATH` than your interactive login shell. A script that calls `aws`, `kubectl`, or `terraform` by bare name often works manually and fails under cron with `command not found`, simply because those tools live in `/usr/local/bin` or `~/.local/bin` and cron's `PATH` doesn't include them. Fix: use absolute paths for critical binaries, or explicitly set `PATH` at the top of the script.

### Shell startup files ⭐

- `/etc/profile`, `~/.bash_profile`, `~/.profile` → loaded for **login** shells
- `~/.bashrc` → loaded for **interactive non-login** shells
- Scripts run via `./script.sh` or cron/systemd load **none of these** by default — that's why aliases and PATH exports in `.bashrc` don't apply to scripts.
- `source file` (or `. file`) executes a file's commands in the *current* shell — used to load libraries/config, not to run a program in a new process.

```bash
source ./lib/logging.sh     # brings functions/vars into current shell
. ./lib/logging.sh           # identical, POSIX form
```

### Exit status, commands, arguments, stdio

Every command returns a numeric **exit status** (0 = success, 1–255 = failure) captured in `$?`. Commands consist of the program name plus **arguments** (`$1`, `$2`, ...). Every process has three standard streams: **stdin** (0), **stdout** (1), **stderr** (2) — covered fully in Part 4.

### Bash execution flow (put it all together)

```
1. Kernel reads shebang → execs bash with your script as input
2. Bash reads a line/command
3. Bash performs expansions in this order:
   brace expansion → tilde expansion → parameter/variable expansion →
   command substitution → arithmetic expansion → word splitting → pathname (glob) expansion
4. Bash resolves the command: alias → function → builtin → $PATH search
5. Bash forks (for external commands) and executes
6. Exit status is stored in $?
7. Bash reads next line
```

Knowing this expansion *order* is what lets you predict bugs like why `$var` with spaces breaks a command, or why `*` in a variable doesn't glob until it's unquoted.

**Common mistakes:** assuming `.bashrc` runs for scripts; assuming `/bin/sh` behaves like Bash; forgetting `chmod +x`; hardcoding tool paths differently between shell and cron.

**Exercises:**
1. Write a one-line script that prints whether it's running as login or non-login (`shopt -q login_shell`).
2. Compare `echo $PATH` from your interactive shell vs. from a cron job (`* * * * * echo $PATH > /tmp/cronpath.txt`).
3. Break a script by using an array in `#!/bin/sh` and observe the error.

---

## PART 2 — Variables ⭐

### Declaration, assignment, expansion

```bash
name="production"        # NO spaces around =
echo $name                # unquoted expansion (risky)
echo "$name"               # quoted expansion (safe) — always prefer this
echo "${name}"              # braced form — required when concatenating: "${name}_backup"
```

`${var}` vs `$var`: functionally identical for simple cases, but `${var}` is required to disambiguate boundaries (`"${name}_v2"` vs the broken `"$name_v2"`, which looks for a variable literally named `name_v2`).

### Quoting basics (full depth in Part 6)

```bash
echo "$HOME is home"        # double quotes: variables & command substitution expand
echo '$HOME is home'         # single quotes: nothing expands, literal text
```

### Command substitution as assignment

```bash
today=$(date +%F)
count=$(wc -l < file.txt)
```

### Local vs environment variables ⭐

- **Shell variable**: exists only in the current shell process.
- **Environment variable**: exported (`export VAR=value`) — inherited by child processes.

```bash
DB_HOST="localhost"          # shell variable — child processes (e.g. a python script called from here) won't see it
export DB_HOST                # now it IS in the environment — children inherit it
export API_KEY="secret"       # export + assign in one line
```

**Production use case 🏭:** Terraform, Ansible, and most CLIs (`aws`, `kubectl`) read config from environment variables (`AWS_PROFILE`, `KUBECONFIG`). Forgetting `export` is a classic bug: the variable is set in your script but invisible to the subprocess you call.

### Read-only and unset variables

```bash
readonly VERSION="1.2.3"     # cannot be reassigned — good for constants
VERSION="2.0.0"                # error: readonly variable

unset DB_HOST                  # removes the variable entirely
```

### Parameter expansion — defaults and error-guards ⭐🏭

This is one of the highest-leverage things to master; it replaces dozens of lines of `if` checks.

```bash
${var:-default}    # use $var if set AND non-empty, else use "default" (does NOT assign)
${var:=default}    # same, but ALSO assigns default to $var if unset/empty
${var:+value}      # if $var IS set/non-empty, substitute "value" (else nothing)
${var:?error msg}  # if $var is unset/empty, print error msg to stderr and exit script
```

```bash
# Real deploy script pattern:
ENVIRONMENT="${1:-staging}"                     # default to staging if no arg given
AWS_REGION="${AWS_REGION:-us-east-1}"            # default region if env var not exported
: "${DEPLOY_KEY:?DEPLOY_KEY must be set}"        # HARD FAIL if secret missing — fail fast
LOG_PREFIX="${VERBOSE:+[verbose] }"               # only add prefix if VERBOSE is set
```

Explanation of the tricky one: `: "${DEPLOY_KEY:?...}"` — the leading `:` is the no-op builtin; it lets you use `${VAR:?msg}` purely for its *side effect* (validating and exiting) without needing to "do" anything with the value.

**Without `:=` vs with:**

```bash
echo "${COUNT:-0}"     # prints 0 if COUNT unset, but COUNT itself stays unset
echo "${COUNT:=0}"     # prints 0 AND sets COUNT=0 for the rest of the script
```

**When to use which:**
- `:-` → provide a fallback for a *one-off read*.
- `:=` → provide a fallback you want *persisted* for later use in the script.
- `:+` → conditionally build strings/flags without an `if`.
- `:?` → guard-clause for required inputs/secrets (fail fast, ⭐ pattern for every production script).

### Variable scope ⭐

```bash
GLOBAL_VAR="I'm global"

my_func() {
    local LOCAL_VAR="I'm local"     # only visible inside this function
    GLOBAL_VAR="modified"            # modifies the global — functions share global scope by default!
}

my_func
echo "$GLOBAL_VAR"     # prints "modified"
echo "$LOCAL_VAR"       # prints nothing — LOCAL_VAR doesn't exist out here
```

**Common mistake 🏭:** forgetting `local` inside functions. In Bash, variables are global by default — a helper function can silently clobber a variable used elsewhere in a large script. Always `local` your function-internal variables; it's the Bash equivalent of variable leakage bugs in other languages.

**Debugging tip:** `declare -p VAR` shows a variable's attributes and value — invaluable when you're not sure if something is set, exported, or an array.

**Exercises:**
1. Write a function that sets a "global-looking" variable without `local`, call it twice, and observe state leaking between calls.
2. Build an env-var validation block for a deploy script using `${VAR:?msg}` for `AWS_REGION`, `ENVIRONMENT`, `IMAGE_TAG`.

**Interview questions:**
- What's the difference between `${var:-x}` and `${var:=x}`? *(one doesn't persist the default, one does)*
- Why must you `export` a variable for a subprocess to see it?
- Why is forgetting `local` in a function dangerous in large scripts?

---

## PART 3 — Bash Special Variables ⭐

| Variable | Meaning |
|---|---|
| `$0` | Name of the script itself |
| `$1`…`$9`, `${10}` | Positional arguments (braces required beyond `$9`) |
| `$#` | Number of positional arguments |
| `$@` | All arguments, as **separate** words |
| `$*` | All arguments, as **one** joined string |
| `$?` | Exit status of the last command |
| `$$` | PID of the current shell/script |
| `$!` | PID of the last background job |
| `$-` | Current shell option flags (e.g. `himBH`) |
| `$_` | Last argument of the previous command |
| `BASH_SOURCE` | Array of source filenames (for functions/sourced files) |
| `BASH_LINENO` | Line numbers corresponding to `FUNCNAME` calls |
| `FUNCNAME` | Array of the current call stack of function names |
| `BASH_VERSION` | Bash's version string |
| `PIPESTATUS` | Array of exit statuses for each stage of the last pipeline |

### `"$@"` vs `"$*"` — the single most important quoting distinction 🏭⭐

```bash
set -- "first arg" "second arg" "third arg"

for a in "$@"; do echo "[$a]"; done
# [first arg]
# [second arg]
# [third arg]

for a in "$*"; do echo "[$a]"; done
# [first arg second arg third arg]     <- ONE item, joined by $IFS (space)
```

**Rule:** when forwarding arguments to another command or function, **always use `"$@"`**, quoted. This is how every well-written wrapper script passes arguments through untouched, including ones with spaces:

```bash
run_kubectl() {
    kubectl --context "$CONTEXT" "$@"     # forwards all CLI args exactly as given
}
run_kubectl get pods -n "my namespace"     # works correctly even with the space
```

If you used `$*` (unquoted or even quoted) here, an argument like `"my namespace"` would get mangled.

### `$?`, `$$`, `$!` in real scripts

```bash
long_running_job &
JOB_PID=$!                        # capture PID of the background job
wait "$JOB_PID"
echo "Job exited with $?"          # check its exit status

echo "This script's PID is $$"     # useful for PID files, temp file naming
```

### `PIPESTATUS` — the fix for pipeline exit-status blindness 🏭🔥

```bash
false | true
echo $?              # prints 0 — misleading! $? only reflects the LAST command in a pipe

false | true
echo "${PIPESTATUS[@]}"    # prints "1 0" — shows every stage's exit code
```

This matters enormously combined with `set -o pipefail` (Part 13) — without it, a failing command early in a pipeline can be silently swallowed.

### `BASH_SOURCE` / `FUNCNAME` / `BASH_LINENO` — for libraries and error traces 🔥

```bash
# lib/utils.sh
get_script_dir() {
    # BASH_SOURCE[0] is reliable even when this file is `source`d from elsewhere,
    # unlike $0 which reflects the CALLING script, not this file.
    cd "$(dirname "${BASH_SOURCE[0]}")" && pwd
}
```

```bash
error_trace() {
    local i=0
    while caller $i; do ((i++)); done
    # prints "lineno function file" for every level of the call stack — great for error() handlers
}
```

**Common mistakes:** using `$*` when forwarding args (breaks on spaces); relying on `$?` right after a pipeline instead of `PIPESTATUS`; confusing `$0` (script name) with `${BASH_SOURCE[0]}` (actual file, correct inside sourced libraries).

**Exercises:**
1. Write a script that receives 3 arguments (one containing spaces) and prints them with both `$@` and `$*` to see the difference.
2. Write a `die()` function using `caller` to print where an error occurred.

**Interview questions:**
- Explain the difference between `"$@"` and `"$*"` with an example that would break if you used the wrong one.
- Why does `$?` alone lie about pipeline failures? How does `PIPESTATUS` fix that?

---

## PART 4 — Input & Output ⭐

### echo vs printf

```bash
echo "Hello, $USER"                 # simple, but portability/flag behavior varies across shells
printf "Hello, %s\n" "$USER"         # POSIX-consistent, supports format specifiers, no surprises
printf "%-10s %5d\n" "cpu" 92         # left-pad string, right-pad number — great for aligned reports
```

**Production preference 🏭:** use `printf` in scripts meant to be portable or produce exact-format output (e.g., CSV, log lines); `echo` is fine for quick human-facing messages.

### read — user input, passwords, multiple variables

```bash
read -p "Enter environment: " ENV                 # -p shows a prompt
read -s -p "Enter password: " PASSWORD; echo        # -s = silent (no echo to terminal)
read -r LINE                                          # -r = don't interpret backslashes (ALWAYS use -r)
read -r NAME AGE CITY <<< "Alice 30 NYC"               # split one line into multiple vars

# Reading with a timeout (useful in interactive automation guards)
if read -t 5 -p "Continue? (y/n) " ans; then
    echo "You said: $ans"
else
    echo "No response in 5s, aborting."
fi
```

`-r` matters: without it, `read` treats backslashes as escape characters, silently corrupting paths like `C:\new\test` or Windows-style input.

### Positional parameters as CLI input

```bash
#!/bin/bash
# usage: ./deploy.sh <environment> <version>
ENVIRONMENT="$1"
VERSION="$2"
echo "Deploying version $VERSION to $ENVIRONMENT"
```

### File descriptors — the real mental model 🏭⭐

Every process starts with three open file descriptors:

| FD | Name | Default destination |
|---|---|---|
| 0 | stdin | keyboard / piped input |
| 1 | stdout | terminal |
| 2 | stderr | terminal |

**Redirection operators:**

```bash
command > file        # stdout → file (overwrite)
command >> file         # stdout → file (append)
command < file            # file → stdin
command 2> file             # stderr → file
command 2>> file              # stderr → file (append)
command 2>&1                    # redirect stderr to WHEREVER stdout currently points
command &> file                   # both stdout AND stderr → file (bash shorthand)
command > /dev/null 2>&1            # discard everything — common in cron/silent automation
```

**Order matters — a classic gotcha:**

```bash
command > file 2>&1     # CORRECT: stdout goes to file, then stderr follows stdout to file
command 2>&1 > file       # WRONG: stderr goes to terminal (current stdout), THEN stdout redirected to file
```

Why: redirections are processed left-to-right. `2>&1` means "point fd 2 at whatever fd 1 currently points to" — the *current* target, not future ones.

**Production pattern — separate logs cleanly 🏭:**

```bash
./deploy.sh > /var/log/deploy.out 2> /var/log/deploy.err
./backup.sh >> /var/log/backup.log 2>&1        # combined append log, standard cron pattern
```

**Common mistakes:** `2>&1 > file` ordering bug (above); forgetting `-r` on `read`; using `echo -n` vs `printf` inconsistently across shells; assuming `>` behaves like `>>` (it truncates!).

**Exercises:**
1. Write a script that logs stdout to one file and stderr to another, then a version that merges them correctly.
2. Build an interactive prompt with `read -t` that falls back to a default after timeout.

---

## PART 5 — Command Substitution ⭐

```bash
current_date=$(date +%F)             # modern syntax — ALWAYS prefer this
current_date2=`date +%F`              # legacy backtick syntax — avoid in new code
```

**Why `$()` is strictly better than backticks:**
- Nesting is clean: `$(dirname "$(realpath "$0")")` vs. unreadable/broken nested backticks (`` `dirname \`realpath $0\`` ``).
- Easier to read, easier to quote correctly, less prone to escaping bugs.
- ShellCheck flags backticks (SC2006) precisely because of this.

### Capturing output and exit status separately

```bash
output=$(some_command)
status=$?                  # $? here reflects `some_command`'s exit status, captured right after

if [[ $status -ne 0 ]]; then
    echo "Command failed: $output" >&2
    exit 1
fi
```

### Common mistakes

```bash
# BUG: word splitting because unquoted
files=$(ls *.log)
for f in $files; do ...     # breaks on filenames with spaces

# FIX:
mapfile -t files < <(find . -name "*.log")
for f in "${files[@]}"; do ...
```

```bash
# BUG: trailing newlines silently stripped is usually fine, but be aware:
count=$(grep -c ERROR app.log)      # returns a string "5", not an integer type — still comparable numerically
```

Always **quote** command substitution when assigning multi-word/whitespace-sensitive results: `result="$(cmd)"`.

**Production use case:** capturing a Git commit SHA, an AWS resource ID, or a `kubectl` JSON field for use later in the script:

```bash
GIT_SHA=$(git rev-parse --short HEAD)
INSTANCE_ID=$(aws ec2 run-instances ... --query 'Instances[0].InstanceId' --output text)
POD_NAME=$(kubectl get pods -l app=web -o jsonpath='{.items[0].metadata.name}')
```

**Exercises:**
1. Rewrite a script full of backticks to use `$()`, including one nested case.
2. Write a function that runs a command, captures both output and exit status, and returns a formatted error if it failed.

---

## PART 6 — Quoting ⭐🏭

This is the #1 source of real production Bash bugs. Master it completely.

### The rules

- **Single quotes `'...'`**: everything literal. No expansion of variables, commands, or globs. Only single quotes themselves can't appear inside.
- **Double quotes `"..."`**: variables (`$var`), command substitution (`$(...)`), and arithmetic (`$((...))`) still expand. Word splitting and glob expansion do **not** happen inside double quotes.
- **Backslash `\`**: escapes the single next character.
- **No quotes**: full word splitting (on `$IFS`, default space/tab/newline) AND glob expansion (`*`, `?`, `[...]`) apply — this is the dangerous default.

### Why `"$variable"` is safer than `$variable`

```bash
filename="my report.txt"

rm $filename        # DANGER: expands to `rm my report.txt` → two args → tries to delete TWO files:
                       # "my" and "report.txt" — neither exists, or worse, something else does!

rm "$filename"        # SAFE: one argument, exactly "my report.txt"
```

### Real bugs caused by bad quoting

**Bug 1 — deleting the wrong thing:**
```bash
find /tmp -name "*.tmp" -exec rm {} \;     # fine
target=$1
rm -rf $target/*                             # if $target is empty/unset, this becomes `rm -rf /*` !!
rm -rf "${target:?}"/*                        # SAFE: fails loudly instead of nuking root
```

**Bug 2 — silent word splitting in a loop:**
```bash
for f in $(find . -name "*.log"); do    # breaks on filenames with spaces or globs
    process "$f"
done
# FIX: use a null-delimited pipeline (see Part 8)
find . -name "*.log" -print0 | while IFS= read -r -d '' f; do
    process "$f"
done
```

**Bug 3 — glob expansion surprise:**
```bash
echo *                    # expands to every file in cwd if unquoted and files exist matching
msg="Files: *"
echo $msg                   # if files happen to match "*" in cwd... no, this is fine actually since * is inside a variable already expanded as literal text UNLESS re-globbed by further unquoted use. The danger is with raw * on command line, not in a variable holding literal "*".
```

**Bug 4 — command injection via unquoted eval-like patterns:** covered fully in Part 26 (Security).

### Practical rule 🏭⭐

> **Quote every variable expansion unless you have a specific, understood reason not to** (e.g., intentionally splitting a list of flags). When in doubt, `"$var"`.

**Debugging technique:** ShellCheck (Part 25) catches almost all quoting bugs automatically — SC2086 is "double quote to prevent globbing and word splitting," and it fires on nearly every unquoted `$var` in a script.

**Exercises:**
1. Create a file called `evil file.txt`, then write a script that fails to delete it due to missing quotes, then fix it.
2. Take any 10-line unquoted script and run it through ShellCheck; fix every SC2086 warning.

**Interview questions:**
- Why is `rm -rf $dir/*` more dangerous than `rm -rf "${dir:?}"/*`?
- What's the practical difference between single and double quotes regarding command substitution?
## PART 7 — Conditionals ⭐

### Syntax

```bash
if [[ condition ]]; then
    ...
elif [[ other_condition ]]; then
    ...
else
    ...
fi
```

### `[ ]` vs `[[ ]]` — know the difference cold 🏭⭐

| | `[ ]` (test) | `[[ ]]` (Bash keyword) |
|---|---|---|
| Portability | POSIX, works in `/bin/sh` | Bash/Zsh/Ksh only |
| Word splitting on unquoted vars | Happens — dangerous | Does not happen — safer |
| `&&` / `\|\|` inside | Must use `-a`/`-o` (deprecated, buggy) | Native `&&`/`\|\|` supported |
| Pattern/regex matching | No | Yes: `[[ $s == pat* ]]`, `[[ $s =~ regex ]]` |
| Empty variable safety | `[ $var = "x" ]` breaks if `$var` is empty/unset | `[[ $var == "x" ]]` is safe even unquoted |

**Rule 🏭:** in Bash scripts (not POSIX `sh` scripts), always prefer `[[ ]]`. Reserve `[ ]` only for portability-required POSIX scripts.

```bash
[[ -z "$var" ]]          # true if var is EMPTY string
[[ -n "$var" ]]           # true if var is NON-empty
[[ "$a" == "$b" ]]         # string equality (use == inside [[, = inside [)
[[ "$a" != "$b" ]]          # string inequality
[[ $num -eq 5 ]]             # numeric equality (use -eq/-ne/-lt/-gt/-le/-ge, NOT ==, for numbers)
[[ -e "$path" ]]               # path exists (any type)
[[ -f "$path" ]]                 # regular file exists
[[ -d "$path" ]]                   # directory exists
[[ -r "$path" ]]                     # readable
[[ -w "$path" ]]                       # writable
[[ -x "$path" ]]                         # executable
[[ -s "$path" ]]                           # exists AND size > 0 (great for "is this backup empty?" checks)
```

### `test` and `(( ))`

```bash
test -f "$path" && echo "exists"      # identical to [ -f "$path" ]

(( count > 10 ))                        # ARITHMETIC context — use for numeric conditions, cleaner than -gt
if (( retries >= max_retries )); then
    echo "Giving up"
fi
```

### `case` (introduced fully in Part 9)

```bash
case "$ENVIRONMENT" in
    prod|production) echo "PRODUCTION — extra caution" ;;
    staging)          echo "staging" ;;
    *)                echo "unknown environment" ;;
esac
```

**Common mistakes:** using `=` for numeric comparisons (`[[ 5 = 05 ]]` is string-false-ish, unpredictable) instead of `-eq`; forgetting quotes inside `[ ]` causing "unary operator expected" errors on empty variables; mixing `-a`/`-o` in `[ ]` (deprecated — use `[[ ]]` with `&&`/`||` instead, or separate `[ ]` calls joined by shell `&&`).

**Exercise:** Write a validation block for a deploy script that checks: version format matches `X.Y.Z` (regex), environment is one of `dev|staging|prod`, and a required config file exists — using `[[ ]]`.

---

## PART 8 — Loops ⭐

### for, while, until

```bash
for env in dev staging prod; do
    echo "Deploying to $env"
done

for ((i=0; i<10; i++)); do            # C-style — useful for numeric counters
    echo "Iteration $i"
done

count=0
while (( count < 5 )); do
    echo "count=$count"
    ((count++))
done

until systemctl is-active --quiet nginx; do
    echo "Waiting for nginx..."
    sleep 2
done
```

### break / continue, nested loops

```bash
for region in us-east-1 us-west-2 eu-west-1; do
    for az in a b c; do
        [[ "$az" == "b" ]] && continue     # skip az "b" in every region
        echo "$region$az"
    done
done
```

### Reading files line-by-line — the ONE correct pattern 🏭⭐

```bash
while IFS= read -r line; do
    echo "Processing: $line"
done < "input.txt"
```

- `IFS=` prevents leading/trailing whitespace from being trimmed.
- `-r` prevents backslash interpretation.
- `< file` feeds the file as stdin to the `while` loop's own subshell — clean, no external process per line.

### Why `for file in $(ls)` is dangerous 🏭⭐

```bash
for file in $(ls); do        # BAD
    process "$file"
done
```

Problems:
1. **Word splitting** — filenames with spaces get split into multiple broken "files."
2. **Glob mangling** — filenames containing `*`, `?`, `[` get glob-expanded unexpectedly.
3. **`ls` output isn't designed for parsing** — its formatting can vary (colorized in some configs, columns when piped in some tools).
4. Fails silently on empty directories or files starting with `-`.

**Safe alternatives:**

```bash
# Option 1 — glob directly (best for simple, single-directory cases)
for file in ./*.log; do
    [[ -e "$file" ]] || continue     # handles the case where no file matches the glob
    process "$file"
done

# Option 2 — find + null-delimited read (best for recursive / untrusted filenames)
while IFS= read -r -d '' file; do
    process "$file"
done < <(find . -name "*.log" -print0)

# Option 3 — mapfile into an array (best when you need the full list, e.g. for a count first)
mapfile -d '' -t files < <(find . -name "*.log" -print0)
echo "Found ${#files[@]} files"
for file in "${files[@]}"; do
    process "$file"
done
```

### Infinite loops & loop control (production: health-check polling)

```bash
while true; do
    if curl -sf "http://localhost/health" > /dev/null; then
        echo "Healthy"
        break
    fi
    sleep 5
done
```

**Common mistakes:** parsing `ls` output; forgetting `IFS=` in `while read`; using `for` over `$(cat file)` instead of `while read`; infinite loops with no `sleep` (busy-loop burning CPU) or no exit condition at all.

**Exercises:**
1. Write a script that safely processes every file in a directory tree, including ones with spaces and special characters, using `find -print0`.
2. Write a polling loop with a maximum attempt count and exponential backoff (foreshadows Part 49).

**Interview question:** Why is `for f in $(ls)` considered a serious anti-pattern? Name three concrete failure modes.

---

## PART 9 — Case Statements ⭐

```bash
case "$1" in
    start)
        systemctl start myapp
        ;;
    stop)
        systemctl stop myapp
        ;;
    restart|reload)                       # multiple patterns, pipe-separated
        systemctl restart myapp
        ;;
    status)
        systemctl status myapp
        ;;
    *)                                       # default/fallback case
        echo "Usage: $0 {start|stop|restart|status}" >&2
        exit 1
        ;;
esac
```

### Real example — deployment script action dispatcher 🏭

```bash
#!/bin/bash
set -euo pipefail

ACTION="${1:-}"
ENVIRONMENT="${2:-staging}"

case "$ACTION" in
    deploy)
        echo "Deploying to $ENVIRONMENT"
        ;;
    rollback)
        echo "Rolling back $ENVIRONMENT"
        ;;
    health-check)
        echo "Checking health of $ENVIRONMENT"
        ;;
    *)
        echo "Usage: $0 {deploy|rollback|health-check} [environment]" >&2
        exit 2
        ;;
esac
```

### CLI command parser with wildcard patterns

```bash
case "$version" in
    v[0-9]*.[0-9]*.[0-9]*) echo "Valid semver-ish: $version" ;;
    *) echo "Invalid version format" >&2; exit 1 ;;
esac
```

### Menu-driven script

```bash
PS3="Select an environment: "
select env in dev staging prod quit; do
    case "$env" in
        quit) break ;;
        dev|staging|prod) echo "Selected: $env"; break ;;
        *) echo "Invalid option" ;;
    esac
done
```

**Common mistakes:** forgetting `;;` (falls through unpredictably — actually causes a syntax error, unlike C's fallthrough); not including a `*)` default case (silent no-op on unmatched input); using `case` for numeric range checks (`[[ ]]`/`(( ))` is clearer there).

**Exercise:** Build a service-management script (`start|stop|restart|status|logs`) around a real systemd unit.

---

## PART 10 — Arrays ⭐

### Indexed arrays

```bash
servers=("web1" "web2" "web3")
echo "${servers[0]}"            # web1
echo "${servers[@]}"              # all elements
echo "${#servers[@]}"               # length: 3

servers+=("web4")                     # append
unset 'servers[1]'                      # remove element (leaves a gap in indices!)

for s in "${servers[@]}"; do echo "$s"; done
```

### Associative arrays (Bash 4+) 🔥

```bash
declare -A instance_types=(
    ["prod"]="m5.large"
    ["staging"]="t3.medium"
    ["dev"]="t3.micro"
)

echo "${instance_types[prod]}"          # m5.large
for env in "${!instance_types[@]}"; do  # !arr[@] = KEYS
    echo "$env -> ${instance_types[$env]}"
done
```

### `${array[@]}` vs `${array[*]}` — same distinction as `$@` vs `$*` 🏭⭐

```bash
arr=("one two" "three")
for x in "${arr[@]}"; do echo "[$x]"; done
# [one two]
# [three]

for x in "${arr[*]}"; do echo "[$x]"; done
# [one two three]     <- collapsed into ONE string
```

**Rule:** always use `"${array[@]}"` (quoted, with `@`) to preserve each element correctly, especially when elements contain spaces.

### Array slicing

```bash
arr=(a b c d e)
echo "${arr[@]:1:3}"     # b c d   (start at index 1, take 3 elements)
echo "${arr[@]: -2}"       # d e     (last two — note the space before -2, required!)
```

### Passing arrays to functions

```bash
process_servers() {
    local -n ref=$1        # nameref — pass array BY REFERENCE (Bash 4.3+)
    for s in "${ref[@]}"; do
        echo "Processing $s"
    done
}
servers=("web1" "web2")
process_servers servers      # pass array NAME, not "${servers[@]}"
```

Alternative (more portable, avoids namerefs): pass elements and rebuild:
```bash
process_servers() {
    local arr=("$@")
    for s in "${arr[@]}"; do echo "$s"; done
}
process_servers "${servers[@]}"
```

**DevOps use case:** collecting a list of unhealthy pods, dirty S3 buckets, or failed CI jobs into an array, then iterating to remediate or report:

```bash
mapfile -t unhealthy_pods < <(kubectl get pods --field-selector=status.phase!=Running -o name)
if (( ${#unhealthy_pods[@]} > 0 )); then
    printf 'Unhealthy: %s\n' "${unhealthy_pods[@]}"
fi
```

**Common mistakes:** using `${arr[*]}` in a loop expecting per-element iteration; forgetting `declare -A` before populating an associative array (silently becomes indexed and breaks); off-by-one errors with slicing; the `${arr[@]: -2}` space requirement (no space = parsed as a different expansion).

**Exercise:** Build an associative array mapping environment → AWS account ID, and a function that validates the current `$ENVIRONMENT` against its keys.

---

## PART 11 — Strings ⭐

Parameter expansion IS Bash's string manipulation toolkit — no external `cut`/`sed` needed for simple cases (faster, no subprocess).

```bash
s="hello-world-example"

echo "${#s}"                 # length: 19

echo "${s:0:5}"                # substring: "hello"      (offset, length)
echo "${s:6}"                    # from index 6 to end: "world-example"
echo "${s: -7}"                    # last 7 chars: "example"   (space required before minus)

echo "${s#hello-}"                   # remove SHORTEST match from FRONT: "world-example"
echo "${s##*-}"                        # remove LONGEST match from FRONT: "example"
echo "${s%-example}"                     # remove SHORTEST match from END: "hello-world"
echo "${s%%-*}"                            # remove LONGEST match from END: "hello"

echo "${s/world/earth}"                      # replace FIRST match: "hello-earth-example"
echo "${s//e/E}"                               # replace ALL matches: "hEllo-world-Example"

echo "${s^^}"                                    # uppercase all:  "HELLO-WORLD-EXAMPLE"
echo "${s,,}"                                      # lowercase all
echo "${s^}"                                         # capitalize first char only

a="hello"; b="world"
echo "$a $b"                                            # concatenation: "hello world"
combined="${a}_${b}"                                      # "hello_world"
```

### Trimming whitespace

```bash
trim() {
    local s="$1"
    s="${s#"${s%%[![:space:]]*}"}"    # strip leading whitespace
    s="${s%"${s##*[![:space:]]}"}"     # strip trailing whitespace
    echo "$s"
}
```

### Splitting and joining

```bash
IFS=',' read -ra parts <<< "us-east-1,us-west-2,eu-west-1"       # split on comma
for region in "${parts[@]}"; do echo "$region"; done

# joining an array with a delimiter:
join_by() { local IFS="$1"; shift; echo "$*"; }
join_by ", " "${parts[@]}"          # "us-east-1, us-west-2, eu-west-1"
```

### Real DevOps use cases 🏭

```bash
# Extract image tag from a full ECR image reference
image="123456789.dkr.ecr.us-east-1.amazonaws.com/my-app:v1.4.2"
tag="${image##*:}"                   # "v1.4.2"
repo="${image%:*}"                     # "123456789.dkr.ecr.us-east-1.amazonaws.com/my-app"

# Strip a file extension
file="backup-2026-09-07.tar.gz"
name="${file%.tar.gz}"                    # "backup-2026-09-07"

# Sanitize a branch name into a valid k8s resource name
branch="feature/JIRA-123_New-Thing"
safe="${branch,,}"                          # lowercase
safe="${safe//[^a-z0-9-]/-}"                  # replace invalid chars with dashes
```

**Common mistakes:** confusing `#`/`##` (front) with `%`/`%%` (end); forgetting the space in `${s: -N}`; reaching for `sed`/`awk` subprocesses for trivial string ops that parameter expansion handles natively and faster.

**Exercises:**
1. Write a function `parse_image()` that splits a Docker image reference into registry, repo, and tag.
2. Write a `slugify()` function usable for Kubernetes resource names.

**Interview question:** What's the difference between `${var#pattern}` and `${var##pattern}`?

---

## PART 12 — Functions ⭐

```bash
greet() {
    local name="$1"
    echo "Hello, $name"
}
greet "World"
```

### Parameters, return values, exit status

```bash
add() {
    local a="$1" b="$2"
    echo $((a + b))            # "return" data via stdout, capture with $()
}
result=$(add 3 4)

is_valid_env() {
    local env="$1"
    [[ "$env" =~ ^(dev|staging|prod)$ ]]     # exit status of [[ becomes the function's return code
}
if is_valid_env "$ENVIRONMENT"; then
    echo "valid"
fi
```

### `return` vs `echo` vs `exit` — critical distinction 🏭⭐

| | Purpose | Scope |
|---|---|---|
| `return N` | Sets the function's **exit status** (0-255, like any command) | Exits the function only |
| `echo "value"` | Sends **data** to stdout, meant to be captured via `$(func)` | Doesn't affect exit status |
| `exit N` | Terminates the **entire script/shell process** | Exits everything, including the caller |

```bash
check_disk() {
    local usage
    usage=$(df / | awk 'NR==2{print $5}' | tr -d '%')
    if (( usage > 90 )); then
        return 1        # signal failure to the CALLER — caller decides what to do
    fi
    return 0
}

if check_disk; then
    echo "disk OK"
else
    echo "disk critical" >&2
    # NOTE: do NOT call exit inside check_disk itself — that would kill the whole script
    # even if the caller wanted to just log a warning and continue.
fi
```

**Rule:** library/helper functions should almost never call `exit` — that removes the caller's ability to decide how to react. Reserve `exit` for the script's main flow or explicit fatal-error handlers.

### Function libraries — reusable scripting building blocks 🏭

```bash
# lib/logging.sh
log_info()  { printf '[%s] INFO: %s\n'  "$(date -Iseconds)" "$*" >&2; }
log_error() { printf '[%s] ERROR: %s\n' "$(date -Iseconds)" "$*" >&2; }
die()       { log_error "$*"; exit 1; }
```

```bash
#!/bin/bash
set -euo pipefail
source "$(dirname "${BASH_SOURCE[0]}")/lib/logging.sh"

log_info "Starting deployment"
[[ -f "config.yaml" ]] || die "config.yaml not found"
```

### Function tracing (debug)

```bash
set -x                      # every command AND function call gets traced with PS4 prefix
PS4='+ ${FUNCNAME[0]:-main}:${LINENO}: '
```

**Common mistakes:** calling `exit` inside a reusable function (kills caller unexpectedly); trying to "return" a string with `return` (it only accepts 0-255 integers — use `echo`+capture, or a global/nameref for complex data); forgetting `local` (Part 2).

**Exercises:**
1. Build a `lib/validation.sh` with functions `is_valid_ip`, `is_valid_semver`, `require_command`.
2. Write a function that returns a non-trivial data structure (e.g., multiple values) using a nameref output parameter.

**Interview question:** Why shouldn't a reusable library function call `exit` directly on failure?

---

## PART 13 — Exit Status & Error Handling 🏭⭐

### The fundamentals

```bash
grep "pattern" file.txt
echo $?              # 0 = found, 1 = not found, 2 = error (e.g. file missing) — check the docs, codes vary by tool!
```

`0` = success. `1–255` = failure, meaning defined by each individual program (not standardized beyond 0=success).

### && and || for control flow

```bash
mkdir -p /app/releases/v1 && echo "created" || echo "failed to create"

command1 && command2      # command2 runs ONLY if command1 succeeds
command1 || command2        # command2 runs ONLY if command1 FAILS
```

### `set -e`, `set -u`, `set -o pipefail`, `set -x` — individually

```bash
set -e         # exit immediately if any command exits non-zero
set -u         # error on use of an undefined variable (catches typos like $ENVIRONEMNT)
set -o pipefail  # a pipeline's exit status = the LAST non-zero command in it, not just the final stage
set -x         # print each command before executing it (debug tracing)
set +x         # turn tracing back off
```

### `set -euo pipefail` in depth 🏭⭐ — THE production header

```bash
#!/bin/bash
set -euo pipefail
IFS=$'\n\t'          # (optional, hardens word-splitting to newlines/tabs only, not spaces)
```

**What each flag actually buys you:**
- `-e`: a script that would otherwise silently continue after a failed command now stops. Prevents "step 3 failed but step 4, 5, 6 ran anyway and made things worse."
- `-u`: catches variable name typos immediately instead of silently treating them as empty strings (which is how `rm -rf "$TARGDIR"/*` becomes `rm -rf /*` when `TARGDIR` was actually misspelled `TARGET_DIR`).
- `-o pipefail`: makes `cmd1 | cmd2` fail the script if `cmd1` fails, even though `cmd2` (e.g. `grep`, `tee`) might return 0.

### Limitations and common pitfalls of `set -e` 🔥🏭

`set -e` does **NOT** trigger in these situations — you must know them or you'll get a false sense of safety:

1. **Inside `if`/`while`/`until` conditions** — a failing command there is *expected* to fail sometimes, so `-e` is suspended:
   ```bash
   set -e
   if grep -q "ERROR" file.log; then    # grep returning 1 (not found) does NOT exit the script here
       echo "found"
   fi
   ```
2. **Commands in a pipeline other than the last, without `pipefail`** — already covered above.
3. **Commands whose result is used with `&&`/`||`** — the failure is "handled," so `-e` doesn't fire:
   ```bash
   risky_command || true      # explicit escape hatch — "I know this can fail, ignore it"
   ```
4. **Inside a function called as part of a condition**, and various corner cases with command substitution in older Bash versions (`local var=$(cmd_that_fails)` historically didn't trigger `-e` — fixed in modern Bash but still worth testing).
5. `set -e` does **not** propagate failure detection through `$(...)` capturing unless you check `$?` explicitly right after in some Bash versions — always verify behavior on your target Bash version.

**Defensive scripting pattern 🏭:**

```bash
set -euo pipefail
trap 'echo "Error on line $LINENO. Exit code: $?" >&2' ERR

log_error() { printf 'ERROR: %s\n' "$*" >&2; }
die() { log_error "$*"; exit 1; }

require_command() {
    command -v "$1" >/dev/null 2>&1 || die "$1 is required but not installed"
}

require_command aws
require_command kubectl
```

### Error propagation across a real script

```bash
deploy() {
    build_image || { log_error "build failed"; return 1; }
    push_image  || { log_error "push failed";  return 1; }
    apply_manifests || { log_error "apply failed"; return 1; }
}

if ! deploy; then
    die "Deployment failed — see logs above"
fi
```

**Common mistakes:** assuming `set -e` catches everything (it doesn't — see the 5 pitfalls above); using `set -e` without testing edge cases in CI; forgetting that a function's `return` value inside `if func; then` suspends `-e` for that call.

**Exercises:**
1. Write a script demonstrating a case where `set -e` does NOT stop execution (inside an `if` condition), then show the safe pattern.
2. Add a global `ERR` trap that logs the failing line number and command to every script in a small "lib."

**Interview questions (very common in real interviews):**
- Explain `set -euo pipefail` line by line, with an example of what each flag prevents.
- Give a concrete example where `set -e` does NOT stop a script, and explain why.
- How do you get the real exit status of the first failing command in a pipeline?

---

## PART 14 — Traps 🏭🔥

```bash
trap 'command' SIGNAL
```

### Common signals to trap

| Signal | Trigger | Typical use |
|---|---|---|
| `EXIT` | Script exits, for ANY reason (normal, error, or signal) | Cleanup — always fires, most reliable |
| `ERR` | Any command returns non-zero (when `set -e` is active) | Error logging/alerting |
| `INT` | Ctrl+C (SIGINT) | Graceful cancel |
| `TERM` | `kill <pid>` (SIGTERM, the default kill signal) | Graceful shutdown |
| `HUP` | Terminal closed / SIGHUP | Reload config or ignore |
| `DEBUG` | Before every command | Fine-grained tracing |
| `RETURN` | When a function or sourced script returns | Function-level tracing |

### Production example — guaranteed temp-file cleanup 🏭⭐

```bash
#!/bin/bash
set -euo pipefail

TMP_DIR=$(mktemp -d)
cleanup() {
    local exit_code=$?
    rm -rf "$TMP_DIR"
    exit "$exit_code"       # preserve the original exit code after cleanup
}
trap cleanup EXIT           # fires no matter HOW the script ends — success, error, or signal

echo "Working in $TMP_DIR"
# ... do work that might fail ...
```

This single pattern (`mktemp` + `trap cleanup EXIT`) is the standard, correct way to guarantee no leftover temp files/directories in production automation, regardless of how the script terminates.

### Graceful shutdown on SIGTERM (long-running daemon-style scripts)

```bash
SHOULD_STOP=0
handle_term() {
    echo "Received SIGTERM, finishing current task then stopping..."
    SHOULD_STOP=1
}
trap handle_term TERM INT

while (( SHOULD_STOP == 0 )); do
    do_work
    sleep 1
done
echo "Stopped gracefully"
```

### ERR trap for structured error reporting

```bash
trap 'log_error "Failed at line $LINENO: $BASH_COMMAND (exit $?)"' ERR
```

`$BASH_COMMAND` holds the literal command that was about to run/just failed — extremely useful in error notifications (Slack/PagerDuty webhooks in production scripts).

**Common mistakes:** forgetting `trap ... EXIT` cleans up even on success (people sometimes think it's only for errors); overwriting a previous `trap` on the same signal (only the last one registered wins — use a trap-stacking pattern if you need multiple handlers); not preserving/re-emitting the original exit code inside a cleanup handler.

**Exercises:**
1. Write a backup script that creates a temp working directory, and guarantees cleanup via `trap ... EXIT` even if the backup command fails mid-way.
2. Write a script that responds to SIGINT by finishing its current loop iteration before exiting, printing "Shutting down gracefully."

**Interview question:** Why is `trap cleanup EXIT` generally better than putting cleanup code at the bottom of the script?

---

## PART 15 — File & Directory Automation ⭐

```bash
mkdir -p /app/releases/"$VERSION"      # -p: no error if exists, creates parents too
touch "$LOCKFILE"
cp -a source/ destination/                # -a: archive mode, preserves permissions/timestamps/symlinks
mv old_name new_name
rm -f "$FILE"                                # -f: don't error if missing (use carefully!)
```

### Checking existence, permissions, ownership

```bash
[[ -e "$path" ]] && echo "exists"
[[ -f "$path" ]] && echo "is a regular file"
[[ -d "$path" ]] && echo "is a directory"
[[ -r "$path" ]] && echo "readable"
[[ -w "$path" ]] && echo "writable"
[[ -x "$path" ]] && echo "executable"
stat -c '%U:%G %a' "$path"          # owner:group and octal permissions
```

### Disk usage

```bash
df -h /                          # filesystem-level usage
du -sh /var/log/*                  # per-directory size, sorted manually or with `sort -h`
```

### Temporary files — always use `mktemp` 🏭⭐

```bash
TMP_FILE=$(mktemp)                        # e.g. /tmp/tmp.XXXXXXXXXX — unpredictable name, avoids collisions
TMP_DIR=$(mktemp -d)                        # temp directory
trap 'rm -f "$TMP_FILE"; rm -rf "$TMP_DIR"' EXIT
```

**Never** hand-roll temp filenames like `/tmp/myfile_$$` alone in security-sensitive contexts — predictable names are subject to race conditions (symlink attacks); `mktemp` creates the file atomically and safely.

### find + xargs — batch operations 🏭⭐

```bash
find /var/log -name "*.log" -mtime +30 -print0 | xargs -0 rm -f      # delete logs older than 30 days
find . -name "*.tmp" -print0 | xargs -0 -I{} mv {} /tmp/archive/
find /data -type f -size +100M                                          # find large files
```

`-print0`/`xargs -0` (null-delimited) is the safe pairing — handles filenames with spaces/newlines correctly, unlike plain `find | xargs`.

### Safe usage of destructive commands 🏭🔥

```bash
# Guard clause pattern before any destructive op:
: "${TARGET_DIR:?TARGET_DIR must be set}"
[[ -d "$TARGET_DIR" ]] || die "TARGET_DIR does not exist: $TARGET_DIR"
[[ "$TARGET_DIR" != "/" ]] || die "Refusing to operate on root"

rm -rf "${TARGET_DIR:?}"/*      # the :? guard here means even a coding mistake that unsets TARGET_DIR
                                   # will error loudly instead of silently expanding to `rm -rf /*`
```

```bash
# "dry run" pattern — extremely valuable for anything destructive in production
DRY_RUN="${DRY_RUN:-true}"
run() {
    if [[ "$DRY_RUN" == "true" ]]; then
        echo "[DRY RUN] $*"
    else
        "$@"
    fi
}
run rm -rf "$TARGET_DIR"
```

**Common mistakes:** `rm -rf $var/*` without the `:?` guard; using `find ... -exec rm {} \;` when `-delete` or `-print0 | xargs -0 rm` is safer/faster for bulk deletes; not checking `df` before writing large files (disk-full mid-operation corruption).

**Exercises:**
1. Write a log-rotation script that finds and compresses logs older than 7 days, and deletes compressed logs older than 90 days — safely, with dry-run support.
2. Write a "safe rm" wrapper function that refuses to operate on `/`, `$HOME`, or an empty variable.

---

## PART 16 — Text Processing ⭐🏭

### grep vs sed vs awk — when to use which

| Tool | Best for |
|---|---|
| `grep` | Finding/filtering lines matching a pattern |
| `sed` | Simple line-based transformation/substitution (stream editor) |
| `awk` | Column/field-based processing, calculations, structured reports |
| `cut` | Extracting fixed columns/delimited fields (simpler than awk for that one job) |

```bash
grep -E "ERROR|CRITICAL" app.log                    # extended regex, multiple patterns
grep -c "ERROR" app.log                                # count matching lines
grep -v "DEBUG" app.log                                  # invert match — exclude lines
grep -r "TODO" ./src                                       # recursive search
grep -A3 -B1 "Exception" app.log                              # 3 lines after, 1 before match — context

sed 's/foo/bar/' file.txt                                   # replace first occurrence per line
sed 's/foo/bar/g' file.txt                                     # replace ALL occurrences per line
sed -i.bak 's/old/new/g' config.conf                             # in-place edit, with a .bak backup
sed -n '10,20p' file.txt                                           # print only lines 10-20

awk '{print $1, $3}' access.log                                        # print columns 1 and 3
awk -F, '{sum += $2} END {print sum}' data.csv                             # sum column 2 of a CSV
awk '$9 == 500 {print $7}' access.log                                        # filter by field value (HTTP 500s)

cut -d',' -f2,4 data.csv                                                        # extract fields 2 and 4
sort -t, -k2 -n data.csv                                                          # numeric sort by field 2
uniq -c                                                                              # count consecutive duplicates
tr 'a-z' 'A-Z' < file.txt                                                              # translate chars
head -n 20 file.txt; tail -n 20 file.txt; tail -f app.log                                # first/last/follow
wc -l file.txt                                                                              # count lines
```

### Real-world example — Nginx log analysis 🏭

```bash
# Top 10 IPs by request count
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head -10

# Count HTTP status codes
awk '{print $9}' access.log | sort | uniq -c | sort -rn

# Requests per minute in the last hour, for a spike investigation
awk '{print $4}' access.log | cut -d: -f1-3 | uniq -c | tail -60
```

### Combining tools in a pipeline

```bash
grep "ERROR" app.log | awk -F'|' '{print $2}' | sort | uniq -c | sort -rn
```

**When grep/sed/awk vs jq/python:** grep/sed/awk shine for line-oriented plain-text logs; once data is structured (JSON/YAML), reach for `jq`/`yq` (Part 36) instead of regex-hacking JSON with `sed` — that's fragile and a common source of parsing bugs.

**Common mistakes:** `sed -i` without a backup extension on production files (risky if the regex is wrong); using `awk '{print $2}'` on data with variable whitespace without setting the right field separator (`-F`); forgetting `sort` before `uniq` (uniq only collapses *consecutive* duplicates).

**Exercises:**
1. Write a one-liner pipeline that reports the top 5 error messages in a log file by frequency.
2. Build a small Nginx access-log analyzer script producing: top IPs, status code breakdown, and slowest requests (if response time is logged).

**Interview question:** When would you reach for `awk` instead of `grep`+`cut`? Give a concrete example.
## PART 17 — Regular Expressions 🔥

### Basic vs Extended regex

```bash
grep "a.c" file.txt              # BRE (basic regex) — default grep
grep -E "a.c|x.z" file.txt         # ERE (extended regex) — enables |, +, ?, {} without backslashes
```

| Symbol | Meaning |
|---|---|
| `.` | any single character |
| `*` | zero or more of the previous element |
| `+` (ERE) | one or more |
| `?` (ERE) | zero or one |
| `^` `$` | anchors: start / end of line |
| `[abc]` | character class |
| `[^abc]` | negated character class |
| `{n,m}` | quantifier: n to m repetitions |
| `(...)` | grouping/capturing (ERE without backslash) |
| `\|` (ERE) | alternation |

### In Bash's `[[ =~ ]]` operator 🏭

```bash
if [[ "$version" =~ ^v?[0-9]+\.[0-9]+\.[0-9]+$ ]]; then
    echo "Valid semver"
fi

if [[ "$ip" =~ ^([0-9]{1,3}\.){3}[0-9]{1,3}$ ]]; then
    echo "Looks like an IPv4 address (format-only, not range-validated)"
fi

# Capturing groups via BASH_REMATCH
if [[ "$log_line" =~ ^([0-9-]+)\ ([0-9:]+)\ (.*)$ ]]; then
    date_part="${BASH_REMATCH[1]}"
    time_part="${BASH_REMATCH[2]}"
    message="${BASH_REMATCH[3]}"
fi
```

### Practical examples

```bash
# Extract IPs from a log file
grep -oE '([0-9]{1,3}\.){3}[0-9]{1,3}' access.log

# Match a valid email (good-enough, not RFC-complete)
grep -E '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$'

# Extract URLs from text
grep -oE 'https?://[^ ]+' notes.txt

# Match log severity + timestamp
awk 'match($0, /^\[[0-9-]+ [0-9:]+\] (ERROR|WARN)/) {print}' app.log

# Validate server hostname convention: web-prod-01, db-staging-02
[[ "$hostname" =~ ^[a-z]+-(prod|staging|dev)-[0-9]{2}$ ]]

# Extract semantic version components
if [[ "$tag" =~ ^v([0-9]+)\.([0-9]+)\.([0-9]+)$ ]]; then
    major="${BASH_REMATCH[1]}"; minor="${BASH_REMATCH[2]}"; patch="${BASH_REMATCH[3]}"
fi
```

**Common mistakes:** forgetting `-E` and getting confused why `+`/`{}`/`|` don't work in `grep`; quoting the regex pattern in `[[ =~ ]]` (don't quote the pattern itself if it contains bracket expressions meant to be interpreted — quoting turns parts of it literal in some Bash versions); overusing regex for structured data (JSON/YAML) instead of `jq`/`yq`.

**Exercises:**
1. Write a regex to extract all 5xx status codes and their URLs from an Nginx access log.
2. Write a hostname-validation function using `[[ =~ ]]` matching your org's naming convention.

---

## PART 18 — Process Management ⭐🏭

```bash
long_task &            # run in background
BG_PID=$!                 # capture its PID
wait "$BG_PID"               # block until it finishes
echo "Exit status: $?"

jobs                           # list background jobs of current shell
fg %1                            # bring job 1 to foreground
bg %1                              # resume job 1 in background

ps aux | grep nginx
pgrep -f "myapp.py"                    # find PID(s) by process name/pattern
pkill -f "myapp.py"                      # kill by pattern
kill -TERM "$PID"                          # graceful
kill -9 "$PID"                               # force kill (SIGKILL, last resort — no cleanup runs)
```

### Process substitution `<(...)` / `>(...)` 🔥

```bash
diff <(kubectl get cm app-config -o yaml) <(cat local-config.yaml)     # compare live vs local, no temp files
comm -23 <(sort list1.txt) <(sort list2.txt)                              # items only in list1
```

### Scripts that manage a service lifecycle 🏭

```bash
is_running() { pgrep -f "myapp" > /dev/null 2>&1; }

start_app() {
    if is_running; then
        echo "Already running"
        return 0
    fi
    nohup /opt/myapp/bin/myapp > /var/log/myapp.log 2>&1 &
    echo $! > /var/run/myapp.pid
    echo "Started with PID $(cat /var/run/myapp.pid)"
}

stop_app() {
    if [[ -f /var/run/myapp.pid ]]; then
        kill "$(cat /var/run/myapp.pid)"
        rm -f /var/run/myapp.pid
    fi
}

restart_if_dead() {
    if ! is_running; then
        echo "$(date -Iseconds): app is down, restarting" >> /var/log/watchdog.log
        start_app
    fi
}
```

**Common mistakes:** `kill -9` as a first resort (skips graceful shutdown/cleanup, `trap` handlers never run); forgetting to remove stale PID files; using `ps aux | grep foo | grep -v grep` instead of the cleaner `pgrep -f foo`.

**Exercises:**
1. Write a watchdog script (for cron/systemd timer) that restarts a crashed process and logs the event.
2. Build a `diff`-based drift detector comparing a live Kubernetes ConfigMap to a file in Git, using process substitution.

---

## PART 19 — Parallel & Concurrent Bash 🔥🏭

```bash
for host in "${hosts[@]}"; do
    check_host "$host" &          # launch in background — runs concurrently
done
wait                                 # wait for ALL background jobs to finish
```

### Concurrency limits (avoid overwhelming a system/API)

```bash
MAX_JOBS=5
for host in "${hosts[@]}"; do
    ((++active >= MAX_JOBS)) && wait -n     # wait for ANY one job to finish before launching more (Bash 4.3+)
    check_host "$host" &
done
wait
```

### GNU parallel (when available) — cleaner than manual job control

```bash
printf '%s\n' "${hosts[@]}" | parallel -j 10 check_host {}
```

### Race conditions & locking with `flock` 🏭⭐

**The problem:** two cron/CI-triggered instances of the same script running simultaneously can corrupt shared state (e.g., both writing to the same file, both deploying at once).

```bash
LOCKFILE="/var/run/myscript.lock"
exec 200>"$LOCKFILE"                  # open FD 200 against the lock file
if ! flock -n 200; then                 # -n: non-blocking — fail immediately if already locked
    echo "Another instance is already running. Exiting."
    exit 1
fi
# ... critical section — guaranteed exclusive ...
# lock auto-releases when the script exits (FD closes)
```

```bash
flock -w 30 200 || { echo "Could not acquire lock within 30s"; exit 1; }   # blocking with timeout
```

### PID files as a simpler (weaker) alternative

```bash
PIDFILE=/var/run/myscript.pid
if [[ -f "$PIDFILE" ]] && kill -0 "$(cat "$PIDFILE")" 2>/dev/null; then
    echo "Already running (PID $(cat "$PIDFILE"))"
    exit 1
fi
echo $$ > "$PIDFILE"
trap 'rm -f "$PIDFILE"' EXIT
```

`flock` is strictly more robust than PID files (atomic, kernel-managed, self-cleaning on process death); prefer it for anything production-critical.

**Common mistakes:** using `wait` without `-n`/job limits and accidentally launching hundreds of parallel SSH/API calls (rate-limited or DoS-like); forgetting `flock` entirely and getting duplicate cron/CI runs corrupting state; not handling partial failures across parallel jobs (one host's `&` job failing doesn't stop the others — you must collect exit statuses).

**Exercise:** Write a script that SSHes into 20 servers in parallel (max 5 concurrent), collects each result, and reports failures — protected by `flock` so it can't run twice simultaneously.

---

## PART 20 — File Descriptors & Advanced I/O 🔥

```bash
exec 3< input.txt              # open FD 3 for reading from input.txt
read -r line <&3                 # read a line via FD 3
exec 3<&-                          # close FD 3

exec 4> output.txt              # open FD 4 for writing
echo "data" >&4
exec 4>&-

exec > logfile.txt 2>&1          # redirect ALL subsequent script output (stdout+stderr) to a file
echo "this goes to logfile.txt now"
```

### Named pipes (FIFOs)

```bash
mkfifo /tmp/mypipe
producer > /tmp/mypipe &
consumer < /tmp/mypipe
```

### Process substitution recap

```bash
while IFS= read -r line; do echo "$line"; done < <(kubectl logs mypod)
```

### Here documents and here strings (full detail in Part 21)

```bash
cat <<EOF
Multi-line text with $variable expansion
EOF

grep "pattern" <<< "$single_string"     # here-string
```

**Real-world use case:** `exec > logfile 2>&1` at the top of a script is a common pattern for "log everything this script does" without redirecting every individual command.

**Common mistakes:** forgetting to close manually-opened file descriptors (`exec 3<&-`) in long-running scripts, leaking FDs over time; confusing `<()`  (process substitution, feeds a command's output as if it were a file) with `<<` (heredoc, inline text block).

---

## PART 21 — Here Documents 🏭

```bash
cat <<EOF > /etc/nginx/sites-available/myapp.conf
server {
    listen 80;
    server_name ${DOMAIN};
    root ${APP_ROOT};
}
EOF
```

Variables expand by default in a heredoc. **Quote the delimiter to disable expansion** — useful for generating literal shell scripts or templates with `$`-containing content you don't want interpreted:

```bash
cat <<'EOF' > generated_script.sh
#!/bin/bash
echo "This \$variable stays literal because EOF is quoted"
EOF
```

### Here strings

```bash
grep "ERROR" <<< "$log_line"          # feed a single variable/string as stdin, no temp file needed
```

### SSH automation with heredocs 🏭

```bash
ssh user@host <<'ENDSSH'
sudo systemctl restart nginx
sudo systemctl status nginx --no-pager
ENDSSH
```

```bash
# With LOCAL variable expansion sent to the remote host:
ssh user@host <<EOF
export APP_VERSION="${APP_VERSION}"
cd /opt/app && ./deploy.sh
EOF
```

### SQL execution via heredoc

```bash
psql -U postgres -d mydb <<SQL
SELECT count(*) FROM users WHERE active = true;
SQL
```

### YAML/config generation

```bash
cat <<EOF > deployment-config.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ${APP_NAME}
spec:
  replicas: ${REPLICA_COUNT}
EOF
```

**Common mistakes:** forgetting to quote the delimiter when generating literal scripts (accidentally letting local `$variables` leak into remote/generated content); indentation issues (heredoc content is NOT trimmed by default — use `<<-EOF` with leading **tabs**, not spaces, to allow indentation).

**Exercise:** Write a script that generates an Nginx virtual host config from a heredoc template, then an SSH-based remote deployment using a second heredoc for the remote commands.

---

## PART 22 — Pipelines ⭐🏭

```bash
cat access.log | grep "500" | awk '{print $1}' | sort | uniq -c | sort -rn
```

### Pipeline exit status and `PIPESTATUS` (recap from Part 3, applied)

```bash
set -o pipefail
grep "ERROR" app.log | mail -s "Errors found" ops@company.com
echo "${PIPESTATUS[@]}"     # inspect every stage individually if needed
```

Without `pipefail`, if `grep` finds nothing (exit 1) but `mail` still exits 0, `$?` after the pipeline is 0 — a silent false-positive "success."

### Debugging complex pipelines 🏭

**Technique: build incrementally, verify each stage.**

```bash
cat access.log | head -5                                 # step 1: confirm input looks right
cat access.log | awk '{print $1}' | head -5                # step 2: confirm extraction
cat access.log | awk '{print $1}' | sort | head -5           # step 3: confirm sort
cat access.log | awk '{print $1}' | sort | uniq -c | sort -rn | head -10   # final
```

**Technique: `tee` to inspect intermediate data without breaking the pipeline:**

```bash
cat access.log | grep "500" | tee /tmp/500s.log | wc -l
```

### Performance considerations

- Every pipe stage that's an *external* program (`grep`, `awk`, `sed`) forks a new process — cheap individually, but a pipeline inside a tight loop (thousands of iterations) is expensive. Prefer parameter expansion / a single `awk` script over many small piped commands inside loops.
- Prefer one well-written `awk` over `grep | awk | sed | cut` chains — fewer forks, fewer synchronization points.

**Common mistakes:** trusting `$?` after a multi-stage pipeline without `pipefail`; excessive process forking in hot loops; unnecessary `cat file | grep pattern` instead of `grep pattern file` (a classic, harmless but wasteful anti-pattern known as "Useless Use of Cat").

**Exercise:** Take a 6-stage pipeline, add `pipefail`, deliberately break one middle stage, and confirm the script now correctly detects the failure via `$?`.

---

## PART 23 — Command-Line Argument Parsing ⭐🏭

### Positional basics

```bash
echo "Script: $0, args: $#, all: $@"
shift            # discard $1, shift $2->$1, $3->$2, etc. — used to walk through args one at a time
```

### `getopts` — the standard way to parse short flags 🏭⭐

```bash
#!/bin/bash
usage() {
    echo "Usage: $0 -e <environment> -v <version> [-d] [-h]"
    exit 1
}

DRY_RUN=false
while getopts ":e:v:dh" opt; do
    case "$opt" in
        e) ENVIRONMENT="$OPTARG" ;;
        v) VERSION="$OPTARG" ;;
        d) DRY_RUN=true ;;
        h) usage ;;
        \?) echo "Invalid option: -$OPTARG" >&2; usage ;;
        :) echo "Option -$OPTARG requires an argument" >&2; usage ;;
    esac
done
shift $((OPTIND - 1))          # remove parsed options, leaving remaining positional args in $@

: "${ENVIRONMENT:?-e environment is required}"
: "${VERSION:?-v version is required}"
```

### Long options (`--environment production`) — manual parsing 🏭🔥

`getopts` doesn't support long options natively. The standard pattern:

```bash
#!/bin/bash
usage() {
    cat <<EOF
Usage: $0 --environment <env> --version <ver> [--dry-run] [--help]
EOF
    exit 1
}

DRY_RUN=false
while [[ $# -gt 0 ]]; do
    case "$1" in
        --environment) ENVIRONMENT="$2"; shift 2 ;;
        --version)     VERSION="$2"; shift 2 ;;
        --dry-run)     DRY_RUN=true; shift ;;
        --help)        usage ;;
        *) echo "Unknown option: $1" >&2; usage ;;
    esac
done
```

### Building a professional CLI tool: `deploy.sh --environment production --version 1.2.3` 🏭

```bash
#!/bin/bash
set -euo pipefail

SCRIPT_NAME=$(basename "$0")
ENVIRONMENT=""
VERSION=""
DRY_RUN=false

usage() {
    cat <<EOF
${SCRIPT_NAME} — deploy an application version to an environment

Usage:
  ${SCRIPT_NAME} --environment <env> --version <ver> [--dry-run]

Options:
  --environment   Target environment: dev | staging | prod   (required)
  --version       Semantic version to deploy, e.g. 1.2.3       (required)
  --dry-run       Print actions without executing them
  -h, --help      Show this help message

Examples:
  ${SCRIPT_NAME} --environment production --version 1.2.3
  ${SCRIPT_NAME} --environment staging --version 1.3.0-rc1 --dry-run
EOF
}

while [[ $# -gt 0 ]]; do
    case "$1" in
        --environment) ENVIRONMENT="$2"; shift 2 ;;
        --version)     VERSION="$2"; shift 2 ;;
        --dry-run)     DRY_RUN=true; shift ;;
        -h|--help)     usage; exit 0 ;;
        *) echo "Unknown option: $1" >&2; usage; exit 1 ;;
    esac
done

[[ -z "$ENVIRONMENT" ]] && { echo "Error: --environment is required" >&2; usage; exit 1; }
[[ -z "$VERSION" ]] && { echo "Error: --version is required" >&2; usage; exit 1; }
[[ "$ENVIRONMENT" =~ ^(dev|staging|prod)$ ]] || { echo "Error: invalid environment" >&2; exit 1; }
[[ "$VERSION" =~ ^[0-9]+\.[0-9]+\.[0-9]+(-[a-zA-Z0-9]+)?$ ]] || { echo "Error: invalid version format" >&2; exit 1; }

echo "Deploying version $VERSION to $ENVIRONMENT (dry_run=$DRY_RUN)"
```

**Common mistakes:** forgetting `shift $((OPTIND - 1))` after `getopts` (leftover positional args aren't cleaned up); not validating required args are actually set after parsing; poor/missing `--help`/usage output (a real professional-CLI expectation).

**Exercises:**
1. Extend the `deploy.sh` above with a `--rollback` flag and a `--region` option defaulting to `us-east-1`.
2. Rewrite it to also accept short flags (`-e`, `-v`) alongside the long ones.

**Interview question:** Why doesn't `getopts` support `--long-options` natively, and what's the standard workaround?

---

## PART 24 — Debugging Bash 🏭⭐

```bash
bash -x script.sh                   # trace every command as it executes, with expansions shown
bash -n script.sh                     # syntax check ONLY — doesn't execute anything
bash -v script.sh                       # print each line as read, before expansion (less common)

set -x        # turn tracing on mid-script
set +x        # turn tracing off
```

### Custom trace prefix (`PS4`) — massively improves `-x` readability 🔥

```bash
export PS4='+ ${BASH_SOURCE[0]}:${LINENO}:${FUNCNAME[0]:-main}(): '
set -x
```

Now each traced line shows exactly which file, line, and function it came from — essential once scripts span multiple sourced libraries.

### Redirecting trace output separately (`BASH_XTRACEFD`) 🔥

```bash
exec 5> /tmp/trace.log
BASH_XTRACEFD=5
set -x
```

Keeps `-x` trace noise out of your normal stdout/stderr — useful when debugging a script whose actual output you also need to inspect cleanly.

### Debug logging pattern

```bash
DEBUG="${DEBUG:-0}"
debug() { [[ "$DEBUG" == "1" ]] && echo "[DEBUG] $*" >&2; }
debug "ENVIRONMENT=$ENVIRONMENT VERSION=$VERSION"
```

### Troubleshooting "works manually, fails in production" 🏭⭐ (extremely common real scenario)

Systematic checklist:
1. **PATH differences** — cron/systemd have minimal `PATH`; check with `which` inside vs. outside the automation context.
2. **Environment variables** — interactive shells load `.bashrc`/`.bash_profile`; cron/systemd do not. Explicitly export everything the script needs, or source an env file.
3. **Working directory** — cron/systemd may run from `/` or `/root`, not where you expect; always `cd` explicitly or use absolute paths.
4. **User/permissions** — cron often runs as a different user than your interactive session; check file ownership and sudo requirements.
5. **TTY-dependent behavior** — `read` prompts, colorized output, or `sudo` needing a password will hang non-interactively.
6. **Locale/timezone differences** — `date`, sorting, and number formatting can differ under a different `LANG`/`TZ`.
7. **stdin availability** — a script that unintentionally reads from stdin (e.g. a stray `read`) will hang or misbehave when there's no terminal attached.

```bash
# Add this diagnostic block temporarily to any "works manually, fails in cron" script:
{
  echo "USER: $(whoami)"
  echo "PWD: $(pwd)"
  echo "PATH: $PATH"
  echo "SHELL: $SHELL"
} >> /tmp/debug_env.log
```

**Common mistakes:** debugging in production without isolating `-x` output; not testing scripts under the actual execution context (cron/systemd/CI) before relying on them; assuming interactive-shell behavior mirrors non-interactive.

**Exercises:**
1. Take a script that references a tool via bare name (e.g. `terraform`) and reproduce a `command not found` failure by simulating cron's minimal PATH (`env -i PATH=/usr/bin:/bin ./script.sh`).
2. Add a custom `PS4` and `bash -x` trace to a multi-function script and read the output.

---

## PART 25 — ShellCheck 🏭⭐

### Why it matters

ShellCheck is a static analyzer purpose-built for shell scripts. It catches quoting bugs, unsafe patterns, portability issues, and common logic errors — the majority of this course's "common mistakes" sections are literally ShellCheck's built-in rule set. **No production Bash script should ship without passing ShellCheck.**

### Installation

```bash
# Debian/Ubuntu
sudo apt-get install shellcheck
# macOS
brew install shellcheck
# Or run via Docker, no install needed:
docker run --rm -v "$PWD:/mnt" koalaman/shellcheck:stable myscript.sh
```

### Reading ShellCheck output

```bash
shellcheck deploy.sh

# In deploy.sh line 12:
rm -rf $TARGET_DIR/*
       ^-- SC2115 (warning): Use "${var:?}" to ensure this never expands to /* (potentially destructive).
```

### Key warning codes to know 🏭

| Code | Meaning | Fix |
|---|---|---|
| SC2086 | Unquoted variable → word splitting/globbing | Quote it: `"$var"` |
| SC2046 | Unquoted command substitution | Quote it: `"$(cmd)"` |
| SC2155 | `declare`/`local` masks a command substitution's exit status | Split into two lines: `local x; x=$(cmd)` |
| SC2164 | `cd` without error handling | `cd "$dir" || exit` |
| SC2068 | Unquoted array expansion (`$@`-style bug on arrays) | Use `"${arr[@]}"` |
| SC2181 | Checking `$?` indirectly instead of directly | `if cmd; then` instead of `cmd; if [[ $? -eq 0 ]]` |

**SC2155 explained (a subtle, common one):**
```bash
local result=$(risky_command)      # BAD: `local` swallows the exit status of the command substitution
echo $?                              # this reflects `local`'s exit status (usually 0), NOT risky_command's!

local result
result=$(risky_command)             # GOOD: split into two lines
echo $?                               # now correctly reflects risky_command's exit status
```

**SC2181 explained:**
```bash
grep "ERROR" file.log
if [[ $? -eq 0 ]]; then echo "found"; fi     # works, but indirect and fragile

if grep -q "ERROR" file.log; then echo "found"; fi   # cleaner, direct, preferred
```

### Integrating ShellCheck into CI/CD 🏭

```yaml
# .github/workflows/lint.yml
name: Lint Bash Scripts
on: [push, pull_request]
jobs:
  shellcheck:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run ShellCheck
        uses: ludeeus/action-shellcheck@master
        with:
          scandir: './scripts'
```

Or via raw CLI in any CI runner:
```bash
find . -name "*.sh" -print0 | xargs -0 shellcheck
```

**Common mistakes:** ignoring ShellCheck warnings without understanding them (use `# shellcheck disable=SC2086` sparingly and only with a comment explaining why); not running it in CI (so regressions slip back in); treating it as optional rather than a required gate for production scripts.

**Exercise:** Run ShellCheck against every script you've written in this course so far and fix every warning, understanding *why* each fix matters rather than blindly applying it.

**Interview question:** What does SC2155 catch, and why is it dangerous specifically for error-handling code?

---

## PART 26 — Security 🏭🔥

### Command injection

```bash
# DANGEROUS: user input concatenated into a command
user_input="somefile; rm -rf /"
eval "cat $user_input"          # executes BOTH `cat somefile` AND `rm -rf /` — catastrophic

# eval is almost never necessary and should be avoided:
eval "$user_input"                # NEVER do this with any untrusted input
```

**Why `eval` is dangerous:** it re-parses its argument as shell code, meaning any shell metacharacters in that string (`;`, `|`, `` ` ``, `$()`, `&`) execute as actual shell syntax, not literal text. This is the shell equivalent of SQL injection.

**Safe alternatives:**
```bash
# If you need dynamic variable names, use namerefs or associative arrays instead of eval:
declare -n ref="$dynamic_var_name"     # nameref — safe, no re-parsing of arbitrary strings
echo "$ref"

# If you need to run a command built from parts, use an array, NOT a string + eval:
cmd=(cat "$filename")
"${cmd[@]}"
```

### Unsafe variable expansion / unquoted variables

Already covered extensively (Part 6) — this is the most common real-world Bash security bug, because unquoted variables allow word splitting that can inject unintended arguments or, combined with command substitution, unintended commands.

### Temporary file vulnerabilities

```bash
# BAD: predictable name, race condition (attacker can pre-create/symlink it)
TMPFILE="/tmp/myapp_$$"

# GOOD: mktemp creates atomically with a random, unpredictable name and correct permissions
TMPFILE=$(mktemp)
```

### Path traversal

```bash
# BAD: user-controlled path used directly
filename="$1"
cat "/var/data/$filename"          # user could pass "../../etc/passwd"

# GOOD: validate/normalize
filename=$(basename "$1")            # strips any directory traversal components
realpath_check=$(realpath -m "/var/data/$filename")
[[ "$realpath_check" == /var/data/* ]] || die "Invalid path"
```

### Secrets handling 🏭⭐

```bash
# BAD: secret visible in process list (ps aux shows all args to every running process!)
curl -H "Authorization: Bearer $API_TOKEN" https://api.example.com     # actually args aren't usually
                                                                          # the leak vector for env-based
                                                                          # tokens, but CLI flags ARE:
mysql -u root -pMySecretPassword                                          # BAD — visible in `ps aux`

# GOOD: read secrets from files or a secrets manager, avoid putting them in argv
mysql --defaults-extra-file=/etc/mysql/creds.cnf
export MYSQL_PWD  # still imperfect but not visible in ps aux like a CLI arg

# BETTER (production): pull secrets at runtime from a vault / AWS Secrets Manager, never store in scripts
DB_PASSWORD=$(aws secretsmanager get-secret-value --secret-id prod/db --query SecretString --output text)
```

**Never** hardcode secrets in scripts committed to Git. Never `echo` a secret to logs. Use `set +x` around any block that handles secrets if tracing is enabled elsewhere in the script.

```bash
set +x                                # ensure tracing is off before secret-handling code
DB_PASSWORD=$(fetch_secret)
set -x                                 # only re-enable after, if needed
```

### File permissions & privilege escalation

```bash
chmod 600 /etc/myapp/secrets.env         # secrets files: owner read/write only
chmod 700 /opt/scripts/deploy.sh            # scripts touching secrets: owner-only execute

# sudo usage — be explicit and minimal, never blanket `sudo bash script.sh` if avoidable:
sudo -u appuser /opt/app/run.sh              # run as the intended non-root user, not root
```

### Input validation & whitelisting 🏭⭐

```bash
# Whitelist, don't blacklist — this is the general security principle
case "$ENVIRONMENT" in
    dev|staging|prod) : ;;                     # explicitly allowed values only
    *) die "Invalid environment: $ENVIRONMENT" ;;
esac

[[ "$INSTANCE_ID" =~ ^i-[0-9a-f]{8,17}$ ]] || die "Invalid instance ID format"
```

**Common mistakes:** using `eval` for "convenience" dynamic behavior; passing secrets as CLI arguments instead of env vars/files; predictable temp filenames; trusting any user/API input without validating format and range.

**Exercises:**
1. Find and fix every instance of unquoted variables, `eval`, and predictable temp files in a script you wrote earlier in this course.
2. Write an input-validation library function `require_match()` that whitelists a value against a regex or fails loudly.

**Interview questions:**
- Why is `eval "$user_input"` dangerous? Give a concrete exploit example.
- How would you securely pass a database password to a script running under cron?

---

## PART 27 — SSH Automation 🏭

```bash
ssh -o BatchMode=yes -o ConnectTimeout=10 user@host "uptime"     # non-interactive, safe for automation
ssh -o StrictHostKeyChecking=accept-new user@host "cmd"             # auto-accept new host keys (CI use)
```

### SSH config for automation consistency

```
# ~/.ssh/config
Host prod-*
    User deploy
    IdentityFile ~/.ssh/deploy_key
    StrictHostKeyChecking accept-new
    ServerAliveInterval 30
```

### SCP / SFTP / rsync

```bash
scp -r ./build/ user@host:/opt/app/releases/
rsync -avz --delete ./dist/ user@host:/var/www/app/       # --delete: mirror exactly, remove extras
sftp user@host <<EOF
put localfile.tar.gz /remote/path/
EOF
```

### Remote scripts and here documents over SSH

```bash
ssh user@host 'bash -s' < local_script.sh                # runs local_script.sh's contents remotely
ssh user@host "APP_VERSION=$VERSION bash -s" <<'EOF'
echo "Deploying version $APP_VERSION"
cd /opt/app && ./deploy.sh
EOF
```

### Multiple servers / parallel SSH automation 🏭🔥

```bash
hosts=(web1.internal web2.internal web3.internal)
for host in "${hosts[@]}"; do
    ssh -o BatchMode=yes "$host" "sudo systemctl restart myapp" &
done
wait

# With result collection and a concurrency cap using flock-free job control:
declare -A results
for host in "${hosts[@]}"; do
    {
        if ssh -o BatchMode=yes -o ConnectTimeout=5 "$host" "sudo systemctl restart myapp"; then
            echo "$host: OK" >> /tmp/ssh_results.log
        else
            echo "$host: FAILED" >> /tmp/ssh_results.log
        fi
    } &
done
wait
cat /tmp/ssh_results.log
```

**Common mistakes:** interactive `StrictHostKeyChecking` prompts hanging unattended automation (fix: `accept-new` or pre-populated `known_hosts`); not setting `ConnectTimeout` (a dead host hangs the whole script); running SSH loops sequentially when they could be parallelized (and vice versa — overwhelming a fleet with unthrottled parallel SSH).

**Exercises:**
1. Write a fleet-wide rolling-restart script: restart one server, health-check it, only proceed to the next if healthy (sequential, not parallel — a canary-style rollout).
2. Write a parallel SSH fact-gathering script (uptime, disk usage) across N hosts with results aggregated into a table.

---

## PART 28 — Bash + systemd 🏭

```bash
systemctl start myapp
systemctl stop myapp
systemctl restart myapp
systemctl status myapp --no-pager
systemctl is-active --quiet myapp && echo "running"
systemctl is-enabled myapp
journalctl -u myapp -f                    # follow live logs
journalctl -u myapp --since "1 hour ago"
```

### Health-check wrapper around systemd

```bash
check_service() {
    local svc="$1"
    if ! systemctl is-active --quiet "$svc"; then
        echo "$svc is DOWN, attempting restart"
        systemctl restart "$svc"
        sleep 5
        systemctl is-active --quiet "$svc" && echo "$svc recovered" || echo "$svc FAILED to recover"
    fi
}
```

### A Bash script triggered by a systemd service + timer (replacing cron) 🏭🔥

```ini
# /etc/systemd/system/backup.service
[Unit]
Description=Nightly backup

[Service]
Type=oneshot
ExecStart=/opt/scripts/backup.sh
User=backup
```

```ini
# /etc/systemd/system/backup.timer
[Unit]
Description=Run backup.service nightly

[Timer]
OnCalendar=*-*-* 02:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now backup.timer
systemctl list-timers                          # verify scheduling
```

**Why systemd timers over cron in modern infra 🏭:** `Persistent=true` catches up missed runs after downtime (cron doesn't); dependency management (`After=`, `Requires=` on other services); structured logging directly into `journalctl` (no manual log redirection needed); per-service resource limits (`MemoryMax=`, `CPUQuota=`).

**Common mistakes:** forgetting `systemctl daemon-reload` after editing unit files; scripts assuming an interactive environment/HOME variable that systemd's minimal service environment doesn't provide; not setting `User=` and running privileged scripts as root unnecessarily.

**Exercise:** Convert an existing cron backup job into a systemd service + timer, and add automatic restart-on-failure via `Restart=on-failure` for a long-running Bash daemon script.

---

## PART 29 — Bash + Cron 🏭⭐

```
# crontab -e
0 2 * * * /opt/scripts/backup.sh >> /var/log/backup.log 2>&1
*/5 * * * * /opt/scripts/healthcheck.sh
```

### Why scripts that work manually fail under cron 🏭⭐ (the classic real-world issue — expect this in interviews)

1. **Minimal `PATH`** — cron's default `PATH` is often just `/usr/bin:/bin`. Fix: set `PATH=` explicitly at the top of the crontab, or use absolute paths inside the script.
2. **No login/interactive shell env** — `.bashrc`/`.bash_profile`/`.profile` are not sourced. Fix: explicitly `source` a needed env file, or set variables in the crontab itself.
3. **Different working directory** — cron typically starts in the user's home directory, not wherever you were when testing manually. Fix: `cd` explicitly at the top of the script.
4. **No TTY** — anything requiring a terminal (interactive `sudo` password prompts, some `ssh` host-key prompts) will hang or fail silently.
5. **Different `HOME`** — some cron implementations set `HOME` differently or not at all, breaking scripts that rely on `~/.aws/credentials`, `~/.kube/config`, etc.

### Logging cron jobs properly

```bash
0 2 * * * /opt/scripts/backup.sh >> /var/log/backup.log 2>&1
```

Inside the script, prefer timestamped structured lines so a growing log file remains debuggable:
```bash
log() { printf '[%s] %s\n' "$(date -Iseconds)" "$*"; }
```

### Preventing overlapping executions (recap of `flock`, applied to cron) 🏭

```bash
#!/bin/bash
exec 200>/var/run/backup.lock
flock -n 200 || { echo "Backup already running"; exit 1; }
# ... backup logic ...
```

### Failure handling & notifications

```bash
#!/bin/bash
set -euo pipefail
trap 'curl -s -X POST "$SLACK_WEBHOOK" -d "{\"text\":\"Backup FAILED on $(hostname) at line $LINENO\"}"' ERR

# ... backup logic ...
```

Or classic email-based cron alerting (cron emails any output by default if `MAILTO` is set and the job produces stdout/stderr):
```
MAILTO=ops@company.com
0 2 * * * /opt/scripts/backup.sh
```

**Common mistakes:** assuming cron env == interactive env (biggest one, by far); not redirecting output (silent failures with no record); no locking (overlapping runs corrupting shared resources); scheduling in server-local time without accounting for DST/timezone changes on critical jobs.

**Exercises:**
1. Take a script that works when run manually and break it by testing it via `env -i /usr/bin/crontab-simulated-env ./script.sh` (minimal env) — identify and fix every failure.
2. Add `flock`-based overlap prevention and Slack failure notification to a cron backup script.

**Interview question:** A teammate says "the script works when I SSH in and run it, but cron says command not found." Walk through your diagnostic process.

---

## PART 30 — Bash + Git 🏭

```bash
current_branch=$(git rev-parse --abbrev-ref HEAD)
git_sha=$(git rev-parse --short HEAD)
git_sha_full=$(git rev-parse HEAD)

if [[ -n "$(git status --porcelain)" ]]; then
    echo "Working directory has uncommitted changes" >&2
    exit 1
fi

latest_tag=$(git describe --tags --abbrev=0)
```

### Commit/branch validation for CI gates

```bash
if [[ ! "$current_branch" =~ ^(main|release/.+)$ ]]; then
    echo "Deploys only allowed from main or release/* branches" >&2
    exit 1
fi

commit_msg=$(git log -1 --pretty=%B)
if [[ ! "$commit_msg" =~ ^(feat|fix|chore|docs)(\(.+\))?: ]]; then
    echo "Commit message doesn't follow Conventional Commits format" >&2
    exit 1
fi
```

### Git hooks (pre-commit) 🏭

```bash
#!/bin/bash
# .git/hooks/pre-commit
set -euo pipefail

echo "Running ShellCheck on staged .sh files..."
files=$(git diff --cached --name-only --diff-filter=ACM -- '*.sh')
[[ -z "$files" ]] && exit 0

for f in $files; do
    shellcheck "$f" || { echo "ShellCheck failed on $f"; exit 1; }
done
```

### Release automation + changelog generation

```bash
next_version() {
    local last_tag current_type
    last_tag=$(git describe --tags --abbrev=0 2>/dev/null || echo "v0.0.0")
    IFS='.' read -r major minor patch <<< "${last_tag#v}"
    echo "v${major}.${minor}.$((patch + 1))"
}

new_tag=$(next_version)
git tag -a "$new_tag" -m "Release $new_tag"
git push origin "$new_tag"

git log "${last_tag}..HEAD" --pretty=format:"- %s (%h)" > CHANGELOG_entry.md
```

### Deployment scripts gated on Git state

```bash
deploy_from_git() {
    local target_env="$1"
    [[ -z "$(git status --porcelain)" ]] || die "Uncommitted changes present, aborting deploy"
    git fetch origin --tags
    git checkout "$VERSION" || die "Tag $VERSION not found"
    ./deploy.sh --environment "$target_env" --version "$VERSION"
}
```

**Common mistakes:** deploying from a dirty working tree without checking `git status --porcelain`; relying on `git describe` without handling the "no tags yet" edge case; git hooks that aren't executable (`chmod +x .git/hooks/pre-commit`) or aren't distributed to teammates (hooks live outside the repo by default — use a tool like `pre-commit` or a Makefile target to install them).

**Exercise:** Build a `release.sh` that validates a clean working tree, bumps a semver tag based on conventional-commit prefixes since the last tag, generates a changelog, and pushes the tag.
## PART 31 — Bash + Docker 🏭

```bash
docker ps --format '{{.Names}}: {{.Status}}'
docker inspect --format='{{.State.Health.Status}}' mycontainer      # healthcheck status
docker logs --tail 100 -f mycontainer
docker exec mycontainer curl -sf localhost:8080/health
```

### Container lifecycle automation

```bash
restart_unhealthy() {
    local container="$1"
    local health
    health=$(docker inspect --format='{{.State.Health.Status}}' "$container" 2>/dev/null || echo "unknown")
    if [[ "$health" == "unhealthy" ]]; then
        echo "Restarting unhealthy container: $container"
        docker restart "$container"
    fi
}
for c in $(docker ps --format '{{.Names}}'); do
    restart_unhealthy "$c"
done
```

### Image cleanup 🏭

```bash
docker image prune -af --filter "until=168h"      # remove dangling/unused images older than 7 days
docker container prune -f
docker system df                                     # inspect disk usage before/after cleanup
```

### Build & deployment scripts

```bash
#!/bin/bash
set -euo pipefail

IMAGE_NAME="myorg/myapp"
TAG="${1:?Usage: $0 <tag>}"

echo "Building ${IMAGE_NAME}:${TAG}"
docker build -t "${IMAGE_NAME}:${TAG}" -t "${IMAGE_NAME}:latest" .

echo "Running smoke test"
docker run --rm "${IMAGE_NAME}:${TAG}" ./healthcheck.sh || { echo "Smoke test failed"; exit 1; }

echo "Pushing image"
docker push "${IMAGE_NAME}:${TAG}"
docker push "${IMAGE_NAME}:latest"
```

### Docker Compose automation

```bash
docker compose pull
docker compose up -d --remove-orphans
docker compose ps --format json | jq -r '.[] | select(.Health != "healthy") | .Name'

wait_for_healthy() {
    local service="$1" timeout=60 elapsed=0
    while (( elapsed < timeout )); do
        status=$(docker compose ps --format json | jq -r --arg s "$service" '.[] | select(.Service==$s) | .Health')
        [[ "$status" == "healthy" ]] && return 0
        sleep 2; ((elapsed+=2))
    done
    return 1
}
```

### Container backup/monitoring

```bash
docker exec db-container pg_dump -U postgres mydb | gzip > "backup-$(date +%F).sql.gz"

# Monitor container resource usage and alert if over threshold
docker stats --no-stream --format '{{.Name}}: {{.CPUPerc}} {{.MemPerc}}'
```

**Common mistakes:** never pruning images/containers (disk fills up on build hosts); not setting resource limits, causing one runaway container to starve others; using `docker logs` without `--tail` on a huge log (floods terminal/memory); healthchecks that don't actually validate app readiness (just checking the process exists, not that it serves traffic).

**Exercises:**
1. Write a Docker deployment script: build → smoke-test → push → deploy via `docker compose up -d` → health-check → rollback to previous tag on failure.
2. Write a disk-usage guardian: if `docker system df` shows over a threshold, prune images/containers older than N days automatically (with dry-run mode).

---

## PART 32 — Bash + Kubernetes 🏭🔥

```bash
kubectl get pods -n myns -o wide
kubectl get pods -n myns --field-selector=status.phase!=Running
kubectl rollout status deployment/myapp -n myns --timeout=120s
kubectl rollout undo deployment/myapp -n myns
kubectl logs -n myns -l app=myapp --tail=100 --since=10m
kubectl top pods -n myns
```

### `deploy.sh` — Kubernetes rollout with health-check gate 🏭

```bash
#!/bin/bash
set -euo pipefail

NAMESPACE="${1:?namespace required}"
DEPLOYMENT="${2:?deployment name required}"
IMAGE="${3:?image required}"

echo "Setting image for ${DEPLOYMENT} to ${IMAGE}"
kubectl set image "deployment/${DEPLOYMENT}" "${DEPLOYMENT}=${IMAGE}" -n "$NAMESPACE"

echo "Waiting for rollout..."
if ! kubectl rollout status "deployment/${DEPLOYMENT}" -n "$NAMESPACE" --timeout=180s; then
    echo "Rollout failed — rolling back" >&2
    kubectl rollout undo "deployment/${DEPLOYMENT}" -n "$NAMESPACE"
    exit 1
fi
echo "Deployment successful"
```

### `rollback.sh`

```bash
#!/bin/bash
set -euo pipefail
NAMESPACE="${1:?namespace required}"
DEPLOYMENT="${2:?deployment name required}"

kubectl rollout undo "deployment/${DEPLOYMENT}" -n "$NAMESPACE"
kubectl rollout status "deployment/${DEPLOYMENT}" -n "$NAMESPACE" --timeout=120s
```

### `health-check.sh`

```bash
#!/bin/bash
set -euo pipefail
NAMESPACE="$1"
LABEL="$2"

not_ready=$(kubectl get pods -n "$NAMESPACE" -l "$LABEL" -o json \
    | jq -r '.items[] | select(.status.containerStatuses[]?.ready == false) | .metadata.name')

if [[ -n "$not_ready" ]]; then
    echo "Not-ready pods:"
    echo "$not_ready"
    exit 1
fi
echo "All pods matching '$LABEL' are ready"
```

### `cleanup.sh` — batch operations across namespaces

```bash
#!/bin/bash
set -euo pipefail

for ns in $(kubectl get ns -o jsonpath='{.items[*].metadata.name}'); do
    completed=$(kubectl get pods -n "$ns" --field-selector=status.phase=Succeeded -o name)
    if [[ -n "$completed" ]]; then
        echo "Cleaning up completed pods in $ns"
        echo "$completed" | xargs -r kubectl delete -n "$ns"
    fi
done
```

### ConfigMap/Secret automation

```bash
kubectl create configmap app-config --from-file=config.yaml -n myns --dry-run=client -o yaml | kubectl apply -f -
kubectl create secret generic db-creds --from-literal=password="$DB_PASSWORD" -n myns --dry-run=client -o yaml | kubectl apply -f -
```

The `--dry-run=client -o yaml | kubectl apply -f -` pattern is the standard **idempotent** way to create-or-update a resource from a script (covered again in Part 48).

### Multi-cluster scripts

```bash
for ctx in $(kubectl config get-contexts -o name); do
    echo "=== $ctx ==="
    kubectl --context "$ctx" get nodes
done
```

**Common mistakes:** not setting `--timeout` on `rollout status` (script hangs forever on a stuck rollout); forgetting `-r` with `xargs` (errors out on empty input instead of no-op'ing); parsing `kubectl get -o wide` text output instead of `-o json`/`jsonpath`/`jq` (fragile, breaks on column changes); not handling multi-cluster context switching safely in scripts that could accidentally hit the wrong cluster.

**Exercises:**
1. Build a canary-rollout helper: scale a canary deployment to 1 replica, health-check it, then proceed to full rollout or roll back.
2. Write a multi-namespace resource auditor reporting pods without resource limits set (`jq` on `kubectl get pods -o json`).

---

## PART 33 — Bash + AWS 🏭🔥

```bash
aws sts get-caller-identity                  # verify which identity/account you're operating as — ALWAYS check this first in prod scripts
aws ec2 describe-instances --filters "Name=tag:Environment,Values=production" --query 'Reservations[].Instances[].InstanceId' --output text
aws s3 sync ./dist/ s3://my-bucket/releases/"$VERSION"/ --delete
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin "$ECR_REGISTRY"
```

### EC2 inventory script 🏭

```bash
#!/bin/bash
set -euo pipefail
aws ec2 describe-instances \
    --filters "Name=instance-state-name,Values=running" \
    --query 'Reservations[].Instances[].[InstanceId,InstanceType,Tags[?Key==`Name`]|[0].Value,PrivateIpAddress]' \
    --output table
```

### S3 backup script

```bash
#!/bin/bash
set -euo pipefail
BUCKET="${1:?bucket required}"
SRC_DIR="${2:?source dir required}"
DATE=$(date +%F)

aws s3 sync "$SRC_DIR" "s3://${BUCKET}/backups/${DATE}/" --storage-class STANDARD_IA
echo "Backup complete: s3://${BUCKET}/backups/${DATE}/"
```

### Old resource cleanup / cost hygiene

```bash
# Find and optionally delete unattached EBS volumes older than 30 days
aws ec2 describe-volumes --filters "Name=status,Values=available" \
    --query 'Volumes[?CreateTime<=`'"$(date -d '30 days ago' -Iseconds)"'`].VolumeId' --output text
```

### Snapshot/AMI management

```bash
aws ec2 describe-snapshots --owner-ids self \
    --query "Snapshots[?StartTime<='$(date -d '90 days ago' -Iseconds)'].SnapshotId" --output text \
    | xargs -n1 -r aws ec2 delete-snapshot --snapshot-id
```

### AWS Lambda invocation from Bash

```bash
aws lambda invoke --function-name my-processor --payload '{"key":"value"}' --cli-binary-format raw-in-base64-out response.json
cat response.json
```

### Deployment automation with AWS + Bash (ECS example) 🏭

```bash
#!/bin/bash
set -euo pipefail
CLUSTER="${1:?cluster required}"
SERVICE="${2:?service required}"
IMAGE="${3:?image required}"

TASK_DEF=$(aws ecs describe-task-definition --task-definition "$SERVICE" --query 'taskDefinition')
NEW_TASK_DEF=$(echo "$TASK_DEF" | jq --arg IMAGE "$IMAGE" \
    '.containerDefinitions[0].image = $IMAGE | del(.taskDefinitionArn,.revision,.status,.requiresAttributes,.compatibilities,.registeredAt,.registeredBy)')

NEW_ARN=$(aws ecs register-task-definition --cli-input-json "$NEW_TASK_DEF" --query 'taskDefinition.taskDefinitionArn' --output text)
aws ecs update-service --cluster "$CLUSTER" --service "$SERVICE" --task-definition "$NEW_ARN"
aws ecs wait services-stable --cluster "$CLUSTER" --services "$SERVICE"
echo "ECS service updated and stable"
```

**Common mistakes:** not verifying `aws sts get-caller-identity` before running destructive commands against what might be the wrong account/profile; using `--output text` when the result could contain multiple items unexpectedly (breaks downstream parsing) vs. `--output json` + `jq` for anything non-trivial; missing IAM permissions causing partial mid-script failures (test permissions ahead of time, fail fast with clear errors); not handling pagination on `describe-*` calls with large result sets (`--query` alone doesn't paginate — use `--no-paginate` awareness or loop with `NextToken`).

**Exercises:**
1. Build an AWS cost/infra report: EC2 instance count and type breakdown by tag, unattached EBS volumes, and unused Elastic IPs, output as a table.
2. Build a script that verifies the AWS identity/account/region before executing any destructive action, aborting if it doesn't match an expected allow-list.

---

## PART 34 — Bash + Databases 🏭

```bash
# PostgreSQL
psql -U postgres -d mydb -c "SELECT count(*) FROM users;"
psql -U postgres -d mydb -t -A -c "SELECT id FROM users WHERE active=true;"    # -t: no headers, -A: unaligned (script-friendly)
pg_dump -U postgres mydb | gzip > "backup-$(date +%F).sql.gz"
gunzip -c backup.sql.gz | psql -U postgres -d mydb_restore

# MySQL
mysql -u root -p"$MYSQL_PWD" -e "SELECT count(*) FROM orders;" mydb
mysqldump -u root -p"$MYSQL_PWD" mydb | gzip > "mysql-backup-$(date +%F).sql.gz"
```

### Database health check 🏭

```bash
check_postgres() {
    if pg_isready -h "$DB_HOST" -p 5432 -U postgres > /dev/null 2>&1; then
        echo "Postgres OK"
        return 0
    else
        echo "Postgres UNREACHABLE" >&2
        return 1
    fi
}
```

### Automated backup with retention 🏭

```bash
#!/bin/bash
set -euo pipefail
BACKUP_DIR="/var/backups/postgres"
RETENTION_DAYS=14
mkdir -p "$BACKUP_DIR"

FILE="${BACKUP_DIR}/mydb-$(date +%F_%H%M%S).sql.gz"
pg_dump -U postgres mydb | gzip > "$FILE"

# verify the backup isn't empty/corrupt before trusting it
if [[ ! -s "$FILE" ]]; then
    echo "Backup file is empty — backup FAILED" >&2
    rm -f "$FILE"
    exit 1
fi
gzip -t "$FILE" || { echo "Backup file is corrupt" >&2; exit 1; }

find "$BACKUP_DIR" -name "*.sql.gz" -mtime "+${RETENTION_DAYS}" -delete
echo "Backup successful: $FILE"
```

### Exporting query results for reporting

```bash
psql -U postgres -d mydb -t -A -F',' -c "SELECT id,email,created_at FROM users" > users_export.csv
```

**Common mistakes:** not verifying a backup file's integrity (`-s` size check + `gzip -t`) before trusting it — "Backup succeeds but produces a corrupt backup" is a real, common production incident (see Part 51); passing DB passwords as CLI args (visible in `ps aux` — use `PGPASSWORD` env var or a `.pgpass`/`my.cnf` credentials file instead); no retention policy, letting backup storage grow unbounded.

**Exercises:**
1. Write a Postgres backup script with integrity verification, retention cleanup, and Slack notification on failure.
2. Write a "restore test" script that periodically restores the latest backup into a scratch database to prove backups are actually usable (a very real production practice — untested backups are not backups).

---

## PART 35 — Bash + APIs 🏭

```bash
curl -s -X GET "https://api.example.com/status"
curl -s -X POST "https://api.example.com/deployments" \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $API_TOKEN" \
    -d '{"version":"1.2.3","environment":"production"}'
```

### Checking HTTP status codes properly 🏭⭐

```bash
status=$(curl -s -o /tmp/response.json -w "%{http_code}" "https://api.example.com/health")
if [[ "$status" -ne 200 ]]; then
    echo "API returned $status" >&2
    cat /tmp/response.json >&2
    exit 1
fi
```

`-w "%{http_code}"` writes the status code, `-o` sends the body to a file, `-s` silences curl's own progress output — this trio is the standard pattern for scriptable HTTP checks (much more reliable than parsing curl's default output).

### Retry logic with backoff for flaky APIs 🏭🔥

```bash
retry_curl() {
    local url="$1" max_attempts=5 attempt=1 delay=2
    while (( attempt <= max_attempts )); do
        if curl -sf --max-time 10 "$url"; then
            return 0
        fi
        echo "Attempt $attempt failed, retrying in ${delay}s..." >&2
        sleep "$delay"
        ((attempt++))
        ((delay *= 2))          # exponential backoff
    done
    echo "All $max_attempts attempts failed" >&2
    return 1
}
```

### curl + jq for API automation 🏭⭐

```bash
response=$(curl -s "https://api.github.com/repos/org/repo/releases/latest")
tag_name=$(echo "$response" | jq -r '.tag_name')
download_url=$(echo "$response" | jq -r '.assets[0].browser_download_url')

curl -sL -o release.tar.gz "$download_url"
```

### Timeouts

```bash
curl --connect-timeout 5 --max-time 30 "$url"     # separate connection vs total-request timeouts
```

**Common mistakes:** not checking HTTP status codes (assuming any curl exit 0 means "success" — curl exits 0 even on a 404/500 unless you use `-f`); no timeout at all, causing scripts to hang indefinitely on a stuck API; not handling malformed/unexpected JSON responses (validate before parsing, especially with `jq`'s `-e` flag to fail on null/error).

**Exercises:**
1. Write a `retry_curl()`-based deployment trigger against a CI API, with exponential backoff and a maximum wait.
2. Write an API health-check script producing a proper Nagios-style exit code (0/1/2) based on response time and status code.

---

## PART 36 — JSON & YAML Automation 🏭⭐

### `jq` fundamentals

```bash
echo '{"name":"web1","status":"running"}' | jq '.name'                 # "web1"
echo '{"name":"web1","status":"running"}' | jq -r '.name'                # web1 (raw, no quotes — use for scripting!)
echo '[{"id":1},{"id":2}]' | jq '.[].id'                                    # 1 \n 2
echo '{"items":[{"id":1,"ok":true},{"id":2,"ok":false}]}' | jq -r '.items[] | select(.ok==true) | .id'
```

### Parsing AWS CLI / kubectl / API output with jq — real production patterns 🏭⭐

```bash
# Parse AWS CLI output (already covered above, repeated here as the canonical pattern)
aws ec2 describe-instances --output json | jq -r '.Reservations[].Instances[] | "\(.InstanceId) \(.State.Name)"'

# Parse kubectl output
kubectl get pods -o json | jq -r '.items[] | select(.status.phase!="Running") | .metadata.name'

# Process a paginated API response, following a next-page cursor
next=""
while :; do
    resp=$(curl -s "https://api.example.com/items?cursor=${next}")
    echo "$resp" | jq -r '.items[].id'
    next=$(echo "$resp" | jq -r '.next_cursor // empty')
    [[ -z "$next" ]] && break
done
```

### Generating JSON safely (never string-concatenate!) 🏭⭐

```bash
# BAD: manual string building breaks on special characters, quotes, unicode
json="{\"name\":\"$name\",\"env\":\"$env\"}"

# GOOD: let jq build it, so escaping is handled correctly
json=$(jq -n --arg name "$name" --arg env "$env" '{name: $name, env: $env}')
```

### `yq` for YAML (same idea as jq, for YAML) 🔥

```bash
yq '.spec.replicas' deployment.yaml
yq -i '.spec.replicas = 5' deployment.yaml            # in-place update
yq '.metadata.labels.env' deployment.yaml
```

### Configuration file generation

```bash
yq -n '.apiVersion="apps/v1" | .kind="Deployment" | .metadata.name=env(APP_NAME)' > deployment.yaml
```

**Common mistakes:** manually string-concatenating JSON (breaks on quotes/special chars — always use `jq -n --arg`); forgetting `-r` and getting quoted strings where raw text is needed downstream; not handling `null` results from `jq` (use `// empty` or `// "default"` to guard); parsing YAML with regex/`grep`/`sed` instead of `yq` (extremely fragile — YAML's structure isn't line-regular).

**Exercises:**
1. Write a script that fetches a paginated API's full result set into a single JSON array using `jq`.
2. Write a config generator that produces a valid Kubernetes manifest from Bash variables via `yq -n`, avoiding any manual string concatenation.

**Interview question:** Why is manually building a JSON string with `"{\"key\":\"$var\"}"` considered bad practice, and what could go wrong?

---

## PART 37 — Log Automation 🏭

```bash
grep -c "ERROR" app.log
grep -B2 -A5 "FATAL" app.log
awk '/ERROR/{count++} END{print count}' app.log
```

### Log rotation & archiving

```bash
#!/bin/bash
LOG_DIR="/var/log/myapp"
find "$LOG_DIR" -name "*.log" -mtime +1 -exec gzip {} \;
find "$LOG_DIR" -name "*.log.gz" -mtime +90 -delete
```

(In production, prefer the system `logrotate` utility over hand-rolled rotation where possible — but understand the Bash equivalent since you'll sometimes need it for non-standard log locations or custom retention logic.)

### Detecting patterns and alerting 🏭

```bash
#!/bin/bash
set -euo pipefail
LOG="/var/log/nginx/access.log"
THRESHOLD=50

error_count=$(awk '$9 ~ /^5/{c++} END{print c+0}' "$LOG")
if (( error_count > THRESHOLD )); then
    curl -s -X POST "$SLACK_WEBHOOK" -d "{\"text\":\"High 5xx rate: $error_count errors\"}"
fi
```

```bash
# High CPU / memory detection
cpu_idle=$(top -bn1 | grep "Cpu(s)" | awk '{print $8}')
if (( $(echo "$cpu_idle < 10" | bc -l) )); then
    echo "ALERT: CPU usage is very high (idle=${cpu_idle}%)"
fi
```

```bash
# Disk full detection
disk_usage=$(df / | awk 'NR==2{gsub("%","",$5); print $5}')
(( disk_usage > 90 )) && echo "ALERT: disk at ${disk_usage}%"
```

```bash
# Failed systemd service detection
systemctl --failed --no-legend | awk '{print $1}'
```

```bash
# Failed deployment detection (from a deploy log convention)
grep -q "DEPLOY_FAILED" /var/log/deploy.log && echo "Last deployment failed — investigate"
```

**Common mistakes:** alerting on every single error occurrence (alert fatigue — use thresholds/windows); not de-duplicating repeated alerts (add a cooldown); parsing `top`'s output format changes across systems/versions without testing (prefer `/proc/stat` parsing or a monitoring agent for anything critical).

**Exercises:**
1. Build a log-based alerting script that watches for a spike in 5xx errors over a rolling 5-minute window and posts to Slack, with a cooldown to avoid repeat alerts.
2. Build a systemd failed-service watcher that runs every 5 minutes via a timer and emails/Slacks on any failed unit.

---

## PART 38 — Monitoring & Health Check Scripts 🏭⭐

**Design principle:** monitoring scripts should return **meaningful exit codes**, not just print text — this lets them plug into Nagios/Icinga, systemd, or simple cron+alerting without extra glue.

```bash
# Standard convention: 0=OK, 1=WARNING, 2=CRITICAL, 3=UNKNOWN
check_disk() {
    local usage
    usage=$(df / | awk 'NR==2{gsub("%","",$5); print $5}')
    if (( usage >= 95 )); then echo "CRITICAL: disk at ${usage}%"; return 2
    elif (( usage >= 85 )); then echo "WARNING: disk at ${usage}%"; return 1
    else echo "OK: disk at ${usage}%"; return 0
    fi
}
```

```bash
check_load() {
    local load1 cores
    load1=$(awk '{print $1}' /proc/loadavg)
    cores=$(nproc)
    (( $(echo "$load1 > $cores * 2" | bc -l) )) && { echo "CRITICAL: load $load1 on $cores cores"; return 2; }
    echo "OK: load $load1"; return 0
}
```

```bash
check_port() {
    local host="$1" port="$2"
    if timeout 3 bash -c "echo > /dev/tcp/${host}/${port}" 2>/dev/null; then
        echo "OK: $host:$port reachable"; return 0
    fi
    echo "CRITICAL: $host:$port unreachable"; return 2
}
```

```bash
check_http() {
    local url="$1"
    local status
    status=$(curl -s -o /dev/null -w "%{http_code}" --max-time 5 "$url")
    [[ "$status" == "200" ]] && { echo "OK: $url returned 200"; return 0; }
    echo "CRITICAL: $url returned $status"; return 2
}
```

```bash
check_ssl_expiry() {
    local domain="$1" warn_days=14
    local expiry epoch_expiry epoch_now days_left
    expiry=$(echo | openssl s_client -servername "$domain" -connect "${domain}:443" 2>/dev/null \
        | openssl x509 -noout -enddate | cut -d= -f2)
    epoch_expiry=$(date -d "$expiry" +%s)
    epoch_now=$(date +%s)
    days_left=$(( (epoch_expiry - epoch_now) / 86400 ))
    if (( days_left < warn_days )); then
        echo "WARNING: SSL cert for $domain expires in $days_left days"; return 1
    fi
    echo "OK: SSL cert valid for $days_left more days"; return 0
}
```

```bash
check_docker_container() {
    local container="$1"
    local status
    status=$(docker inspect --format='{{.State.Health.Status}}' "$container" 2>/dev/null || echo "missing")
    [[ "$status" == "healthy" ]] && { echo "OK: $container healthy"; return 0; }
    echo "CRITICAL: $container is $status"; return 2
}
```

```bash
check_k8s_pods() {
    local ns="$1" label="$2"
    local not_ready
    not_ready=$(kubectl get pods -n "$ns" -l "$label" --field-selector=status.phase!=Running --no-headers | wc -l)
    (( not_ready > 0 )) && { echo "CRITICAL: $not_ready pods not running"; return 2; }
    echo "OK: all pods running"; return 0
}
```

```bash
check_db() {
    pg_isready -h "$1" -p 5432 -q && { echo "OK: db reachable"; return 0; }
    echo "CRITICAL: db unreachable"; return 2
}
```

**Exercise:** Combine all the checks above into a single `healthcheck.sh --check disk|load|port|http|ssl|docker|k8s|db` dispatcher (using `case`, from Part 9) that a monitoring system can call uniformly.

---

## PART 39 — Backup Automation 🏭

### Complete production backup script

```bash
#!/bin/bash
set -euo pipefail

BACKUP_ROOT="/var/backups/app"
S3_BUCKET="s3://my-backups-bucket"
RETENTION_DAYS=30
DATE=$(date +%F_%H%M%S)
WORKDIR=$(mktemp -d)
LOGFILE="/var/log/backup.log"

trap 'rm -rf "$WORKDIR"' EXIT
exec >> "$LOGFILE" 2>&1

log() { printf '[%s] %s\n' "$(date -Iseconds)" "$*"; }
notify_failure() {
    curl -s -X POST "$SLACK_WEBHOOK" -d "{\"text\":\"Backup FAILED: $1\"}" > /dev/null || true
}
trap 'notify_failure "unexpected error at line $LINENO"' ERR

log "Starting backup"

# 1. File backup with compression
tar -czf "${WORKDIR}/files-${DATE}.tar.gz" /opt/app/data

# 2. Database backup
pg_dump -U postgres mydb | gzip > "${WORKDIR}/db-${DATE}.sql.gz"

# 3. Verify backups aren't empty/corrupt
for f in "${WORKDIR}"/*.gz; do
    [[ -s "$f" ]] || { log "ERROR: $f is empty"; exit 1; }
    gzip -t "$f" || { log "ERROR: $f is corrupt"; exit 1; }
done

# 4. Encrypt (example using gpg, symmetric)
for f in "${WORKDIR}"/*.gz; do
    gpg --batch --yes --passphrase-file /etc/backup/gpg_pass -c "$f"
done

# 5. Upload to S3
aws s3 sync "$WORKDIR" "${S3_BUCKET}/${DATE}/" --exclude "*" --include "*.gpg"

# 6. Local retention cleanup
find "$BACKUP_ROOT" -mtime "+${RETENTION_DAYS}" -delete

# 7. Remote retention cleanup (S3 lifecycle rules are the better long-term approach,
#    but a script-driven cleanup is shown here for full control)
aws s3 ls "${S3_BUCKET}/" | while read -r line; do
    dir_date=$(echo "$line" | awk '{print $2}' | tr -d '/')
    [[ -z "$dir_date" ]] && continue
    if [[ $(date -d "$dir_date" +%s 2>/dev/null || echo 0) -lt $(date -d "-${RETENTION_DAYS} days" +%s) ]]; then
        aws s3 rm "${S3_BUCKET}/${dir_date}/" --recursive
    fi
done

log "Backup completed successfully: ${DATE}"
```

### Restore testing (the step most teams skip — don't)

```bash
#!/bin/bash
set -euo pipefail
LATEST=$(aws s3 ls "$S3_BUCKET/" | sort | tail -1 | awk '{print $2}')
aws s3 sync "${S3_BUCKET}/${LATEST}" /tmp/restore_test/
gunzip -c /tmp/restore_test/db-*.sql.gz | psql -U postgres -d restore_test_db
psql -U postgres -d restore_test_db -c "SELECT count(*) FROM users;"   # sanity-check row counts
```

**Common mistakes:** never testing restores (a backup you've never restored is a hope, not a backup); no integrity verification before declaring success; unbounded retention (storage costs grow forever) or too-aggressive retention (deleting a backup you needed); storing encryption passphrases insecurely (use a secrets manager, not a plaintext file, in real production).

**Exercise:** Add a scheduled restore-test job (weekly) that restores the latest backup into a scratch DB and compares row counts against a baseline, alerting on mismatch.

---

## PART 40 — Deployment Automation 🏭⭐

```
Git → Build → Test → Backup → Deploy → Health Check → Rollback if failure
```

### Full pipeline script

```bash
#!/bin/bash
set -euo pipefail

VERSION="${1:?version required}"
ENVIRONMENT="${2:?environment required}"
LOG="/var/log/deploy-${ENVIRONMENT}.log"

exec > >(tee -a "$LOG") 2>&1
log() { printf '[%s] %s\n' "$(date -Iseconds)" "$*"; }

PREVIOUS_VERSION=$(cat /opt/app/CURRENT_VERSION 2>/dev/null || echo "none")

rollback() {
    log "ROLLING BACK to $PREVIOUS_VERSION"
    ./deploy_artifact.sh "$PREVIOUS_VERSION" "$ENVIRONMENT"
    exit 1
}
trap rollback ERR

log "Step 1: Pre-deployment checks"
[[ -z "$(git status --porcelain)" ]] || { log "Dirty working tree"; exit 1; }
git fetch --tags && git checkout "$VERSION"

log "Step 2: Build"
docker build -t "myapp:${VERSION}" .

log "Step 3: Test"
docker run --rm "myapp:${VERSION}" pytest

log "Step 4: Backup current state"
./backup.sh

log "Step 5: Deploy"
./deploy_artifact.sh "$VERSION" "$ENVIRONMENT"
echo "$VERSION" > /opt/app/CURRENT_VERSION

log "Step 6: Health check"
for i in {1..10}; do
    if curl -sf "http://localhost/health"; then
        log "Health check passed"
        trap - ERR                 # deployment is good — disarm the rollback trap
        exit 0
    fi
    sleep 3
done
log "Health check failed after retries"
exit 1                                # triggers rollback via ERR trap
```

### Blue-green & canary concepts (Bash's role)

- **Blue-green:** Bash orchestrates deploying the new version to an idle ("green") environment, health-checking it fully, then flipping a load balancer/DNS/Ingress target — the switch itself is typically a single fast API call (e.g., `aws elbv2 modify-listener` or updating a Kubernetes Service selector), with instant rollback by flipping back.
- **Canary:** Bash scales a small percentage of traffic (e.g., 1 of 10 pods, or a weighted Ingress rule) to the new version, watches error-rate/latency metrics for a soak period, then proceeds to full rollout or aborts — this is where retry/health-check loops (Parts 38, 49) directly compose with deployment scripts.

```bash
# Canary example: scale canary deployment, watch error rate, decide
kubectl scale deployment/myapp-canary --replicas=1 -n prod
sleep 60   # soak period
error_rate=$(query_prometheus_error_rate myapp-canary)
if (( $(echo "$error_rate > 1.0" | bc -l) )); then
    kubectl scale deployment/myapp-canary --replicas=0 -n prod
    die "Canary error rate too high ($error_rate%), aborted rollout"
fi
kubectl set image deployment/myapp myapp="$IMAGE" -n prod   # promote to full rollout
```

**Common mistakes:** no automatic rollback path (manual rollback under pressure is error-prone); health checks that check "process started" instead of "actually serving correct traffic"; deploying without first confirming the artifact/image was actually built from the intended commit; skipping the backup step "just this once."

**Exercises:**
1. Build the full pipeline script above end-to-end against a toy app, verifying the rollback path actually triggers correctly on a simulated health-check failure.
2. Extend it into a blue-green pattern using two Docker Compose stacks and an Nginx upstream switch.
## PART 41 — Bash Script Architecture 🏭🔥

Beyond a handful of lines, treat a Bash project like real software:

```
project/
├── bin/                # user-facing entry-point scripts (thin — parse args, call lib functions)
├── lib/                # reusable function libraries (logging.sh, aws.sh, validation.sh, retry.sh)
├── config/              # environment configs, defaults.env
├── scripts/               # internal/maintenance scripts not meant as the main CLI
├── tests/                   # bats test files (Part 42)
├── logs/                       # runtime log output (usually gitignored)
└── README.md
```

### Modular structure in practice

```bash
# lib/logging.sh
log_info()  { printf '[%s] INFO: %s\n'  "$(date -Iseconds)" "$*" >&2; }
log_error() { printf '[%s] ERROR: %s\n' "$(date -Iseconds)" "$*" >&2; }
die() { log_error "$*"; exit 1; }
```

```bash
# lib/validation.sh
require_command() { command -v "$1" >/dev/null 2>&1 || die "$1 is required but not installed"; }
require_env() { [[ -n "${!1:-}" ]] || die "$1 must be set"; }
is_valid_semver() { [[ "$1" =~ ^v?[0-9]+\.[0-9]+\.[0-9]+$ ]]; }
```

```bash
# bin/deploy
#!/bin/bash
set -euo pipefail
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
source "${SCRIPT_DIR}/../lib/logging.sh"
source "${SCRIPT_DIR}/../lib/validation.sh"
source "${SCRIPT_DIR}/../config/defaults.env"

require_command kubectl
require_env AWS_PROFILE

log_info "Starting deploy"
# ... orchestration only — actual logic lives in lib/ functions ...
```

**Why this matters 🏭:** thin entry points + reusable libraries means every script gets logging, error handling, and validation for free and consistently; you fix a bug in one place (`lib/`) instead of N scattered scripts; it's testable (Part 42) because logic lives in sourceable functions, not tangled inline in a monolithic file.

**Common mistakes:** copy-pasting the same logging/error-handling boilerplate into every script instead of centralizing it; using relative paths for `source` that break depending on the caller's working directory (fix: resolve via `${BASH_SOURCE[0]}` as shown above); giant 1000-line single-file scripts with no functions at all.

**Exercise:** Refactor 3-4 standalone scripts you've written in this course into a `project/` layout with a shared `lib/`.

---

## PART 42 — Bash Testing 🏭🔥

### Why test Bash at all

Bash scripts run in production, touch infrastructure, and are just as prone to regressions as application code — yet are usually the least-tested part of a codebase. **Bats** (Bash Automated Testing System) is the standard framework.

```bash
# Install: git clone or `brew install bats-core` / apt package `bats`
```

### Example test file

```bash
# tests/validation.bats
setup() {
    source "${BATS_TEST_DIRNAME}/../lib/validation.sh"
}

@test "is_valid_semver accepts a valid version" {
    run is_valid_semver "1.2.3"
    [ "$status" -eq 0 ]
}

@test "is_valid_semver rejects an invalid version" {
    run is_valid_semver "not-a-version"
    [ "$status" -eq 1 ]
}

@test "require_command fails for a missing binary" {
    run require_command "definitely_not_a_real_command"
    [ "$status" -eq 1 ]
    [[ "$output" == *"required but not installed"* ]]
}
```

```bash
bats tests/*.bats
```

### Mocking external commands

```bash
@test "deploy retries on failure and eventually succeeds" {
    # override `curl` with a mock function for this test
    curl() { echo "500"; }
    export -f curl
    run retry_curl "http://fake"
    [ "$status" -eq 1 ]
}
```

### Testing exit codes explicitly

```bash
@test "script exits 2 on invalid environment" {
    run ./bin/deploy --environment badenv --version 1.0.0
    [ "$status" -eq 2 ]
}
```

### CI integration

```yaml
# .github/workflows/test.yml
jobs:
  bats-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: bats-core/bats-action@2.0.0
      - run: bats tests/*.bats
```

**Common mistakes:** only testing the "happy path" (also test invalid input, missing dependencies, and simulated command failures via mocking); not isolating tests from real infrastructure (tests that actually hit AWS/K8s in CI are slow, flaky, and dangerous — mock external commands); skipping tests for "just glue scripts" — glue scripts are exactly what breaks production the most.

**Exercises:**
1. Write Bats tests for the `lib/validation.sh` and `lib/logging.sh` libraries you built in Part 41.
2. Add a GitHub Actions job running `shellcheck` + `bats` on every PR touching `*.sh` files.

---

## PART 43 — Performance Optimization 🔥

### Where Bash performance actually matters

Bash is an interpreted, line-oriented shell — its "slow" operation is **spawning external processes** (`fork`+`exec`), not the interpreter loop itself. Every `grep`, `awk`, `cat`, `sed` call inside a loop costs real time.

```bash
# SLOW: spawns `wc` once per iteration, thousands of times in a big loop
for f in *.log; do
    lines=$(wc -l < "$f")
done

# FASTER: let one process (awk/wc) handle everything in a single pass
wc -l -- *.log
```

```bash
# SLOW: external `basename`/`dirname` calls
name=$(basename "$path")

# FASTER: pure Bash parameter expansion, no subprocess
name="${path##*/}"
```

```bash
# awk vs a Bash loop for line-processing large files — awk wins decisively
# SLOW (Bash read loop doing arithmetic per line on a huge file):
total=0
while IFS=',' read -r _ amount; do
    total=$((total + amount))
done < big_file.csv

# FASTER (awk processes the whole stream in one optimized pass):
awk -F',' '{total += $2} END {print total}' big_file.csv
```

### Pipelines and process spawning

Each `|` stage is a new process; a 6-stage pipeline run 10,000 times in a loop is 60,000 forks. Restructure to do more work per external-command invocation (batch inputs, use `awk`'s full field-processing power) rather than looping with many small commands.

### When Bash is the right tool vs when to switch 🏭⭐

| Use Bash when... | Switch to Python/Go/Ansible/Terraform when... |
|---|---|
| Gluing existing CLI tools together (kubectl, aws, docker, git) | You need real data structures (nested objects, complex state) |
| Simple sequential automation, health checks, deploy orchestration | You need robust JSON/YAML manipulation beyond `jq`/`yq` one-liners |
| Fast prototyping of an ops task | The script exceeds ~300-500 lines or has deep nested logic |
| The task is inherently "run this list of commands" | You need real unit testing, packages, or shared libraries across teams at scale |
| No complex error recovery / retries beyond what traps handle | You need concurrency beyond simple background jobs (real thread/async models) |
| Infra provisioning declaratively | → Terraform, not Bash-wrapped AWS CLI calls, for anything stateful/idempotent by design |
| Configuration management at fleet scale | → Ansible, which already solves idempotency/inventory/templating that Bash reimplements poorly |

**Rule of thumb:** if you find yourself building your own retry framework, your own YAML parser, or your own inventory system in Bash, that's usually a sign the job has outgrown Bash.

**Common mistakes:** micro-optimizing Bash string operations that don't matter, while ignoring the actual bottleneck (subprocess spawning in loops); rewriting an entire orchestration script in Python "for performance" when the actual bottleneck is network calls (kubectl/aws latency), which no language choice fixes.

**Exercise:** Take a Bash script with a `while read` loop calling 3 external commands per line over a 100k-line file, and rewrite the hot path as a single `awk` script; benchmark both with `time`.

---

## PART 44 — Advanced Bash 🔥🚀

```bash
declare -i num=5              # -i: treat as integer, arithmetic context automatically applied
declare -r CONST=10             # -r: readonly (same as `readonly`)
declare -x EXPORTED=val           # -x: export (same as `export`)
declare -a arr                      # -a: indexed array
declare -A assoc                      # -A: associative array
typeset -i num                          # ksh-compatible alias for `declare` (rarely needed in pure Bash)

local var="x"          # function-scoped variable
export VAR="x"           # environment-visible to children
source ./lib.sh            # execute file in current shell context

eval "$cmd"        # AVOID — re-parses string as shell code (Part 26 security)

exec ./other_script.sh    # REPLACES the current process entirely with other_script.sh — no return!
caller                       # shows calling context (file:line) — used inside error traces
builtin cd /tmp                 # force use of the shell builtin, bypassing any function named `cd`
command ls                        # bypass aliases/functions, run the real `ls` binary
type ls                             # shows whether `ls` is aliased, a function, builtin, or a file, and where
hash -r                               # clear Bash's remembered PATH lookup cache (useful after installing new tools mid-session)
compgen -A function                     # list all defined function names — used in tab-completion scripts
```

### mapfile / readarray 🔥

```bash
mapfile -t lines < file.txt              # read entire file into an array, one element per line
mapfile -t pods < <(kubectl get pods -o name)
echo "Total: ${#lines[@]}"
```

### coproc 🚀 (rare, but real use case: bidirectional pipe to a long-lived subprocess)

```bash
coproc MYPROC { python3 -u worker.py; }
echo "task1" >&"${MYPROC[1]}"
read -r result <&"${MYPROC[0]}"
```

Used when you need a persistent subprocess you can send multiple inputs to and read multiple outputs from, without restarting it each time (e.g., a long-lived helper process for repeated computations).

### Namerefs — indirect variable references 🚀

```bash
set_result() {
    local -n outref=$1        # outref is now an alias for whatever variable name was passed in $1
    outref="computed value"
}
my_var=""
set_result my_var
echo "$my_var"                  # "computed value" — function "returned" data via reference, safely (no eval needed)
```

### Indirect expansion (older, `eval`-free way to reference dynamic variable names)

```bash
varname="HOME"
echo "${!varname}"       # prints the VALUE of $HOME — indirection via `!`
```

**Common mistakes:** reaching for `eval` when a nameref or indirect expansion (`${!var}`) would do the same job safely; using `exec script.sh` when you meant to just call it (forgetting `exec` never returns to the calling script); confusing `declare -i` arithmetic context surprises (a variable declared `-i` auto-evaluates arithmetic on every assignment, which can silently coerce strings to `0` in unexpected places).

**Exercises:**
1. Rewrite a script that used `eval` for dynamic variable access to instead use namerefs.
2. Use `mapfile` to load a `kubectl get pods -o name` list into an array and iterate with per-pod error handling.

**Interview question:** What's the difference between `declare -n` (nameref) and `eval` for achieving "dynamic" variable behavior, and why is one considered safe and the other not?

---

## PART 45 — Bash Signals & Job Control 🔥🏭

| Signal | Number | Meaning | Catchable? |
|---|---|---|---|
| SIGHUP | 1 | Terminal/session closed | Yes |
| SIGINT | 2 | Ctrl+C | Yes |
| SIGTERM | 15 | Standard "please stop" (default `kill`) | Yes |
| SIGKILL | 9 | Force kill, immediate | **No** — cannot be trapped, no cleanup runs |
| SIGSTOP | 19 | Pause process | **No** |
| SIGCONT | 18 | Resume a stopped process | Yes |

### Graceful shutdown pattern (recap, expanded) 🏭

```bash
cleanup_and_exit() {
    echo "Received termination signal, cleaning up..."
    kill "$WORKER_PID" 2>/dev/null || true
    wait "$WORKER_PID" 2>/dev/null || true
    exit 0
}
trap cleanup_and_exit SIGTERM SIGINT

long_running_worker &
WORKER_PID=$!
wait "$WORKER_PID"
```

### Process groups & parent/child relationships 🔥

```bash
# By default, killing a script does NOT kill its background children unless you handle it:
myapp &
CHILD_PID=$!
kill "$CHILD_PID"          # kills only that one process

# To kill an entire process GROUP (script + all its children):
kill -- -$$              # negative PID = process group ID, kills the whole group
```

**Production scenario 🏭:** a deploy script spawns several background health-check pollers; if the parent script is killed (e.g., CI job cancelled), those pollers can become orphaned zombie processes unless you `trap` and explicitly kill the process group or track/kill each PID.

```bash
trap 'kill -- -$$ 2>/dev/null' EXIT     # ensure no orphaned children survive this script
```

**Common mistakes:** assuming `kill $$` cleans up child processes automatically (it doesn't — children become orphans, reparented to init/systemd); trying to trap `SIGKILL` (impossible by design — this is why forceful `kill -9` skips all your cleanup logic, and should be a last resort); not distinguishing `SIGTERM` (graceful, expected) from `SIGKILL` (immediate, no chance to clean up) when designing shutdown handlers.

**Exercise:** Write a script that spawns 3 background workers, and on receiving SIGTERM, gracefully stops all of them (not orphaning any), waiting up to 10 seconds before force-killing stragglers.

---

## PART 46 — Portable Shell Scripting 🔥

### POSIX sh vs Bash-specific features

| Feature | POSIX `sh` | Bash |
|---|---|---|
| Arrays | No | Yes |
| `[[ ]]` | No (use `[ ]`) | Yes |
| `${var//x/y}` | No | Yes |
| `local` | Not POSIX-standard (works in most `sh` implementations anyway, but not guaranteed) | Yes |
| `function` keyword | No (`name() { }` form only) | Both forms work |
| Process substitution `<()` | No | Yes |
| `$'...'` ANSI-C quoting | No | Yes |
| `(( ))` arithmetic | No (`sh` uses `expr` or `$(( ))` which IS POSIX) | Yes, plus `$(( ))` |

### When to use `#!/bin/sh` vs `#!/bin/bash` 🏭

- Use **`#!/bin/sh`** when: the script must run in minimal environments (Alpine containers without bash installed, embedded systems, `/etc/init.d` scripts on some distros, Docker `ENTRYPOINT` scripts targeting `alpine` base images), and you deliberately restrict yourself to POSIX-only syntax.
- Use **`#!/bin/bash`** when: you control the target environment (most servers, most CI runners, most non-Alpine containers) and want Bash's much richer feature set — this is the right default for almost all DevOps automation you write for your own infrastructure.

**Real gotcha 🏭:** Alpine-based Docker images use `busybox sh`/`ash`, not Bash, by default. A script with `#!/bin/bash` will fail with "not found" unless you `apk add bash`. This trips people up constantly when writing Dockerfile `RUN`/`ENTRYPOINT` scripts for Alpine images.

```dockerfile
FROM alpine:3.20
RUN apk add --no-cache bash        # needed if your scripts use bash-isms
COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh
ENTRYPOINT ["/entrypoint.sh"]         # entrypoint.sh's own shebang determines the interpreter used
```

**Common mistakes:** writing `#!/bin/sh` but using Bash-only syntax inside (works locally because `/bin/sh` happens to BE bash on your machine, then breaks on a real Debian/Alpine box where `/bin/sh` is `dash`/`ash`); assuming all servers have Bash pre-installed (mostly true for full Linux distros, false for minimal containers).

**Exercise:** Take a Bash script using arrays and `[[ ]]`, run it with `sh script.sh` on a Debian-based system, and observe/document every failure caused by `dash`'s stricter POSIX behavior.

---

## PART 47 — Production Best Practices 🏭⭐ (Master Checklist)

This section consolidates everything above into the standard "production Bash script header/checklist" you should apply to every real script:

```bash
#!/bin/bash
set -euo pipefail                    # fail fast, catch typos, catch pipeline failures
IFS=$'\n\t'                            # harden word splitting (optional but common)

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
source "${SCRIPT_DIR}/lib/logging.sh"

trap 'log_error "Failed at line $LINENO: $BASH_COMMAND"' ERR
trap cleanup EXIT

LOCKFILE="/var/run/$(basename "$0").lock"
exec 200>"$LOCKFILE"
flock -n 200 || { log_error "Already running"; exit 1; }
```

**Checklist 🏭:**
- [ ] `set -euo pipefail` at the top
- [ ] Quote every variable expansion
- [ ] ShellCheck passes clean (in CI)
- [ ] Structured logging (timestamped, to stderr for logs, stdout reserved for real output)
- [ ] Explicit error handling — guard clauses, `die()`, meaningful exit codes
- [ ] Cleanup via `trap ... EXIT`, guaranteed regardless of exit path
- [ ] All required inputs validated (`${VAR:?msg}`, whitelisted `case` checks)
- [ ] Idempotent — safe to re-run (Part 48)
- [ ] Locking (`flock`) if the script must not run concurrently with itself
- [ ] Timeouts on any network/external call
- [ ] Retries with backoff on flaky operations (Part 49)
- [ ] No hardcoded secrets; secrets pulled from a vault/secrets manager at runtime
- [ ] Version-controlled, code-reviewed like any other code
- [ ] Documented usage/`--help`
- [ ] Tested (Bats) for at least the critical paths and failure modes
- [ ] Monitored/alerting on failure (Slack/PagerDuty webhook in error trap)

**Interview question:** If you were reviewing a colleague's production deployment script, what's your top-10 checklist?

---

## PART 48 — Idempotency 🏭⭐

**Idempotent** means: running the script twice (or a hundred times) produces the same end state as running it once — no duplicate resources, no errors on re-run, no drift.

### Why it's critical in DevOps automation

Production automation gets re-run constantly: retried CI jobs, re-triggered deploys, cron jobs that overlap despite locking bugs, manual re-runs after a partial failure. Non-idempotent scripts turn "just run it again" into a dangerous guess.

```bash
# NOT idempotent — fails on second run because the user already exists
useradd deployuser

# IDEMPOTENT — check first, or use a tool that's idempotent by design
id -u deployuser &>/dev/null || useradd deployuser
```

```bash
# NOT idempotent
mkdir /app/releases

# IDEMPOTENT
mkdir -p /app/releases          # -p: no error if it already exists
```

```bash
# NOT idempotent — apt errors if package list needs updating differently, but more subtly:
# installing an already-installed package is usually fine, but pinning matters:
apt-get install -y nginx=1.18.0    # idempotent: re-running is a no-op if already at that exact version

# NOT idempotent — appends every run, growing unboundedly
echo "export PATH=$PATH:/opt/tool/bin" >> ~/.bashrc

# IDEMPOTENT — check before appending
grep -qxF 'export PATH=$PATH:/opt/tool/bin' ~/.bashrc || echo 'export PATH=$PATH:/opt/tool/bin' >> ~/.bashrc
```

```bash
# Docker resources — idempotent creation pattern
docker network inspect mynet &>/dev/null || docker network create mynet
```

```bash
# Kubernetes — kubectl apply is idempotent BY DESIGN (declarative), unlike kubectl create
kubectl apply -f deployment.yaml         # safe to run repeatedly
kubectl create -f deployment.yaml          # NOT idempotent — errors on second run ("already exists")
```

```bash
# AWS resources — check-then-act, or use tags/idempotency tokens where the API supports them
existing=$(aws ec2 describe-security-groups --filters "Name=group-name,Values=my-sg" --query 'SecurityGroups[0].GroupId' --output text)
if [[ "$existing" == "None" ]]; then
    aws ec2 create-security-group --group-name my-sg --description "..."
fi
```

**Why this is critical 🏭⭐:** it's the difference between "safe to automate/retry blindly" and "requires a human to check state before every run." Idempotency is *the* property that makes automation trustworthy at scale — it's also exactly what Terraform/Ansible/Kubernetes give you natively (declarative, converge-to-desired-state), which is part of why Part 43's "when to switch tools" table recommends them for provisioning/config-management instead of hand-rolled Bash.

**Common mistakes:** treating "runs without error the first time" as sufficient testing (always test a *second* run); appending to config files without checking for existing entries; using imperative "create" APIs instead of declarative "apply/ensure" ones where available.

**Exercises:**
1. Take any script from this course that creates a resource (file, directory, AWS resource, K8s object) and make it idempotent; prove it by running it three times in a row with no errors or duplication.
2. Write an idempotent "ensure user + SSH key + sudoers entry" provisioning script.

**Interview question:** Why is `kubectl apply` idempotent while `kubectl create` is not? How does this inform which one you use in a CI/CD pipeline?

---

## PART 49 — Retries, Timeouts & Resilience 🏭🔥

### Reusable retry function with exponential backoff (the canonical pattern) 🏭⭐

```bash
retry() {
    local max_attempts="$1" delay="$2"; shift 2
    local attempt=1
    until "$@"; do
        if (( attempt >= max_attempts )); then
            echo "Command failed after $attempt attempts: $*" >&2
            return 1
        fi
        echo "Attempt $attempt failed, retrying in ${delay}s..." >&2
        sleep "$delay"
        ((attempt++))
        ((delay *= 2))          # exponential backoff
    done
}

retry 5 2 curl -sf "https://api.example.com/health"
retry 3 5 aws s3 cp file.txt s3://bucket/
retry 4 1 ssh deploy@host "systemctl status myapp"
```

### Command timeout

```bash
timeout 30 ./long_running_task.sh              # kill the command if it exceeds 30 seconds
timeout --signal=TERM --kill-after=5 30 ./task.sh   # send TERM at 30s, force KILL if still alive 5s later
```

### Network/API/SSH-specific timeouts (recap, consolidated)

```bash
curl --connect-timeout 5 --max-time 30 "$url"
ssh -o ConnectTimeout=10 "$host" "cmd"
```

### Health checks + resilience together

```bash
wait_for_healthy() {
    local url="$1" max_wait=120 elapsed=0
    while (( elapsed < max_wait )); do
        curl -sf "$url" > /dev/null && return 0
        sleep 5; ((elapsed+=5))
    done
    return 1
}
```

### Circuit-breaker concept 🚀 (rare in pure Bash, but worth understanding)

A circuit breaker stops attempting a known-failing operation for a cooldown period, instead of retrying indefinitely and hammering a dying dependency:

```bash
CIRCUIT_FILE="/tmp/circuit_open"
CIRCUIT_COOLDOWN=300

call_api() {
    if [[ -f "$CIRCUIT_FILE" ]]; then
        local opened_at; opened_at=$(cat "$CIRCUIT_FILE")
        if (( $(date +%s) - opened_at < CIRCUIT_COOLDOWN )); then
            echo "Circuit open, skipping call" >&2
            return 1
        fi
        rm -f "$CIRCUIT_FILE"        # cooldown expired, allow a retry ("half-open" state)
    fi
    if ! curl -sf "https://api.example.com"; then
        date +%s > "$CIRCUIT_FILE"    # trip the circuit
        return 1
    fi
}
```

This is a simplified illustration — in real systems, a proper circuit breaker (with half-open probing, failure-rate thresholds) is usually implemented at the application/service-mesh layer (e.g., Istio, resilience libraries), not hand-rolled in Bash. But knowing the *concept* — stop retrying a known-broken dependency, back off, then cautiously probe again — informs how you design retry logic in scripts that call flaky external systems.

**Common mistakes:** retrying forever with no max attempts (infinite loop that never surfaces the real problem); fixed-delay retries hammering an already-overloaded service (use exponential backoff, always); no timeout at all on the underlying command (a `retry` wrapper around a command that itself hangs forever never actually retries — it just hangs on attempt 1 permanently. **Always combine `retry` with `timeout`.**).

```bash
retry 5 3 timeout 10 curl -sf "https://api.example.com"    # the CORRECT combination
```

**Exercises:**
1. Build a generic `lib/retry.sh` with `retry()` (exponential backoff) and ensure every network call in your course scripts so far is wrapped in `retry ... timeout ...`.
2. Implement the simple circuit-breaker pattern above around a flaky health-check call and demonstrate it "opening" after repeated failures.

**Interview question:** Why must a retry loop always be combined with a timeout on the underlying command? What happens if you forget the timeout?
## PART 50 — Real-World DevOps Projects 🏭🚀

Below are 20 projects, beginner → expert. The first 6 are built out in full (requirements, architecture, code, explanation, failure modes) as templates; the rest are scoped with clear requirements and starting structure so you can build them the same way — ask me any time to expand a specific one to full code.

### Project 1 (Beginner) — System Information Script

**Requirements:** report hostname, OS, kernel, CPU, memory, disk, uptime in a clean format.
**Architecture:** single script, no external state.

```bash
#!/bin/bash
set -euo pipefail
echo "=== System Info: $(hostname) ==="
echo "OS:      $(grep PRETTY_NAME /etc/os-release | cut -d= -f2 | tr -d '\"')"
echo "Kernel:  $(uname -r)"
echo "CPU:     $(nproc) cores"
echo "Memory:  $(free -h | awk '/Mem:/{print $3"/"$2}')"
echo "Disk:    $(df -h / | awk 'NR==2{print $3"/"$2" ("$5" used)"}')"
echo "Uptime:  $(uptime -p)"
```
**Explanation:** each line calls a standard `/proc`-backed tool and extracts the relevant field with `awk`/`cut` — no error handling needed since these commands essentially never fail on a healthy Linux box, but you'd add `set -euo pipefail` regardless as a habit.
**Security:** read-only, no risk. **Testing:** run on different distros, confirm `/etc/os-release` parsing is robust. **Improvement:** add `--json` output mode via `jq -n`.

### Project 2 (Beginner) — Disk Monitoring Script

**Requirements:** alert (exit 2) if any mount exceeds a usage threshold.
```bash
#!/bin/bash
set -euo pipefail
THRESHOLD="${1:-90}"
FAILED=0
while read -r usage mount; do
    usage="${usage%\%}"
    if (( usage >= THRESHOLD )); then
        echo "CRITICAL: $mount at ${usage}%"
        FAILED=1
    fi
done < <(df -h --output=pcent,target | tail -n +2)
exit $FAILED
```
**Failure scenarios:** `df` output format varies slightly across systems — test the `--output` flag is supported (GNU coreutils; not on BSD/macOS). **Production use:** wire into cron + Slack webhook on `exit 1`.

### Project 3 (Beginner) — User Management Script (idempotent)

```bash
#!/bin/bash
set -euo pipefail
USERNAME="${1:?username required}"
SSH_KEY="${2:?path to public key required}"

if ! id -u "$USERNAME" &>/dev/null; then
    useradd -m -s /bin/bash "$USERNAME"
    echo "Created user $USERNAME"
else
    echo "User $USERNAME already exists"
fi

mkdir -p "/home/${USERNAME}/.ssh"
if ! grep -qF "$(cat "$SSH_KEY")" "/home/${USERNAME}/.ssh/authorized_keys" 2>/dev/null; then
    cat "$SSH_KEY" >> "/home/${USERNAME}/.ssh/authorized_keys"
fi
chmod 700 "/home/${USERNAME}/.ssh"
chmod 600 "/home/${USERNAME}/.ssh/authorized_keys"
chown -R "${USERNAME}:${USERNAME}" "/home/${USERNAME}/.ssh"
```
**Security consideration:** validate `$USERNAME` against a safe pattern (`[[ $USERNAME =~ ^[a-z_][a-z0-9_-]*$ ]]`) before using it in filesystem paths — untrusted input here is a path-traversal/injection risk.

### Project 4 (Intermediate) — Log Analyzer (Nginx)

```bash
#!/bin/bash
set -euo pipefail
LOG="${1:?path to access.log required}"

echo "--- Top 10 IPs ---"
awk '{print $1}' "$LOG" | sort | uniq -c | sort -rn | head -10
echo "--- Status Code Breakdown ---"
awk '{print $9}' "$LOG" | sort | uniq -c | sort -rn
echo "--- Top 10 Requested Paths ---"
awk '{print $7}' "$LOG" | sort | uniq -c | sort -rn | head -10
echo "--- 5xx Error Sample ---"
awk '$9 ~ /^5/ {print}' "$LOG" | tail -5
```
**Testing:** validate against real Nginx `combined` log format; **improvement:** support custom log formats via a `--format` flag mapping field positions.

### Project 5 (Intermediate) — Docker Cleanup Tool

```bash
#!/bin/bash
set -euo pipefail
DRY_RUN="${DRY_RUN:-true}"
DAYS="${1:-7}"

run() { [[ "$DRY_RUN" == "true" ]] && echo "[DRY RUN] $*" || "$@"; }

run docker container prune -f --filter "until=${DAYS}0h" 2>/dev/null || true
run docker image prune -af --filter "until=${DAYS}0h"
run docker volume prune -f
echo "Disk usage after cleanup:"
docker system df
```
**Production consideration:** never prune volumes on a host running stateful containers without checking they're truly unused first — add a `--skip-volumes` default-safe flag.

### Project 6 (Intermediate) — SSL Certificate Checker (fleet-wide)

```bash
#!/bin/bash
set -euo pipefail
DOMAINS_FILE="${1:?path to domains list required}"
WARN_DAYS=14

while IFS= read -r domain; do
    [[ -z "$domain" ]] && continue
    expiry=$(echo | openssl s_client -servername "$domain" -connect "${domain}:443" 2>/dev/null \
        | openssl x509 -noout -enddate 2>/dev/null | cut -d= -f2)
    if [[ -z "$expiry" ]]; then
        echo "$domain: COULD NOT CHECK"
        continue
    fi
    days_left=$(( ($(date -d "$expiry" +%s) - $(date +%s)) / 86400 ))
    if (( days_left < WARN_DAYS )); then
        echo "$domain: WARNING — expires in $days_left days"
    else
        echo "$domain: OK ($days_left days left)"
    fi
done < "$DOMAINS_FILE"
```

### Projects 7–20 (scoped — request full build-out any time)

| # | Project | Level | Core requirement |
|---|---|---|---|
| 7 | CPU/memory monitor with alert thresholds | Intermediate | `/proc/stat` delta sampling for accurate CPU%, not just `top` snapshot |
| 8 | Backup automation (files + DB) | Intermediate | Compression, integrity check, retention, S3 upload (Part 39 template) |
| 9 | Service monitoring daemon | Intermediate | systemd timer + `flock` + Slack alert on failed unit |
| 10 | Website health checker (multi-endpoint) | Intermediate | Config-driven list of URLs, HTTP status + latency thresholds, Nagios-style exit codes |
| 11 | Docker deployment script | Intermediate/Advanced | Build → smoke test → push → deploy → health-check → rollback (Part 31 template) |
| 12 | Kubernetes health checker | Advanced | `jq`-based not-ready pod detection across namespaces, Part 32/38 templates |
| 13 | Kubernetes deployment tool | Advanced | Rollout + automatic rollback on failed health check (Part 32/40 template) |
| 14 | AWS EC2 inventory script | Advanced | Tag-filtered, `--output table`, scheduled report emailed weekly |
| 15 | AWS S3 backup automation | Advanced | Sync + lifecycle-aware retention + restore-test job (Part 33/39) |
| 16 | PostgreSQL backup automation | Advanced | Full Part 34 template + automated restore verification |
| 17 | API monitoring tool | Advanced | `retry`+`timeout` wrapped multi-endpoint checker, JSON report output via `jq -n` |
| 18 | Production deployment script | Expert | Full Part 40 pipeline: Git → Build → Test → Backup → Deploy → Health-check → Rollback |
| 19 | Complete DevOps automation CLI | Expert | Single `getopts`/long-flag dispatcher (Part 23) unifying deploy/rollback/health-check/backup as subcommands, built on the `lib/` architecture from Part 41, tested with Bats (Part 42) |
| 20 | Multi-cloud/multi-cluster orchestrator | Expert | Iterates AWS accounts × K8s contexts (Part 32/33), parallelized with concurrency limits (Part 19) and `flock` guarding against overlapping runs |

For every project you build, apply the **Part 47 production checklist** — that consistency is what separates "a script that works" from "a script you can trust in production."

---

## PART 51 — Real Production Incidents 🏭🔥

Format: **Symptoms → Investigation → Commands → Root Cause → Fix → Prevention**

**1. Script deletes the wrong files**
- *Symptoms:* Unexpected files missing after a "cleanup" script ran.
- *Investigation:* Check the script's `rm` calls for unquoted/unguarded variables.
- *Commands:* `grep -n "rm " script.sh`, `shellcheck script.sh`
- *Root cause:* `rm -rf "$TARGET_DIR"/*` where `$TARGET_DIR` was empty/unset due to an upstream typo, expanding to `rm -rf /*`.
- *Fix:* `rm -rf "${TARGET_DIR:?}"/*`, add a path allow-list check before any destructive op.
- *Prevention:* ShellCheck in CI (catches this as SC2115); mandatory dry-run mode for destructive scripts; never run destructive automation as root unless strictly necessary.

**2. Deployment script partially succeeds**
- *Symptoms:* App is deployed but config/migrations weren't applied — inconsistent state.
- *Investigation:* Re-run with `set -x`; check where it stopped.
- *Commands:* `bash -x deploy.sh 2>&1 | tee trace.log`
- *Root cause:* No `set -e`, so a failed migration step didn't stop the deploy from "succeeding" — the next step ran anyway.
- *Fix:* Add `set -euo pipefail`; make each stage explicitly checked (`|| { rollback; exit 1; }`).
- *Prevention:* Full pipeline pattern from Part 40, with an ERR trap triggering rollback.

**3. Backup succeeds but produces a corrupt backup**
- *Symptoms:* Restore attempt fails; backup file exists but is truncated/unreadable.
- *Investigation:* Check disk space at backup time; check the backup process's exit code, not just file existence.
- *Commands:* `gzip -t backup.sql.gz`, `df -h` (historical, via monitoring)
- *Root cause:* Disk filled up mid-`pg_dump`, truncating output; the script only checked "file exists," not integrity or exit status of `pg_dump`.
- *Fix:* Check `pg_dump`'s exit status directly (not just the file), verify with `gzip -t`, and check file size (`-s`) before declaring success (Part 34/39 pattern).
- *Prevention:* Regular restore testing; disk-space pre-check before starting backup; alert on backup size deviating significantly from historical average.

**4. Cron script works manually but fails automatically**
- *Root cause:* Minimal cron `PATH`/environment (Part 29).
- *Fix:* Absolute paths or explicit `PATH=`/sourced env file in the script.
- *Prevention:* Always test scripts via `env -i` simulation before relying on cron; log `$PATH`/`whoami`/`pwd` at script start temporarily when diagnosing.

**5. SSH automation fails intermittently**
- *Root cause:* No `ConnectTimeout`, hanging on a partially-reachable host; or interactive host-key prompt blocking non-interactively.
- *Fix:* `-o BatchMode=yes -o ConnectTimeout=10 -o StrictHostKeyChecking=accept-new`.
- *Prevention:* Standardize SSH options via `~/.ssh/config` or a wrapper function used everywhere.

**6. AWS CLI authentication fails in CI but works locally**
- *Root cause:* CI uses a different `AWS_PROFILE`/role than the developer's local session; expired/rotated credentials; missing `AWS_REGION`.
- *Fix:* `aws sts get-caller-identity` as the first line of any AWS-touching script — fail fast with a clear message.
- *Prevention:* Use short-lived roles via OIDC in CI (e.g., GitHub Actions' AWS OIDC integration) instead of static keys; validate identity before every run.

**7. Kubernetes deployment gets stuck**
- *Root cause:* `rollout status` called without `--timeout`, the script hangs indefinitely on a stuck pod (image pull failure, resource limits).
- *Fix:* Always set `--timeout`; on timeout, automatically `rollout undo`.
- *Prevention:* Readiness/liveness probes correctly configured; resource requests/limits set to avoid scheduling failures.

**8. Script hangs indefinitely**
- *Root cause:* Stray `read` command with no input redirected consuming from an unexpected stdin (e.g., inside a `while read` loop that also calls `ssh` without `-n`, which itself consumes stdin).
- *Fix:* `ssh -n` inside loops reading from stdin; explicit timeouts (`timeout N cmd`) on anything potentially blocking.
- *Prevention:* Always test scripts in a genuinely non-interactive context (`< /dev/null`) before deploying to cron/CI.

**9. Infinite loop**
- *Root cause:* A polling `while` loop missing a max-attempt/timeout guard, waiting on a condition that will never become true (e.g., waiting for a service that crashed and never restarts).
- *Fix:* Always bound polling loops (`max_wait`, attempt counters) as shown throughout Part 38/49.
- *Prevention:* Code review checklist item: "every wait loop must have a maximum bound."

**10. Race condition — two scripts run simultaneously**
- *Root cause:* No locking; overlapping cron runs corrupted a shared state file.
- *Fix:* `flock` (Part 19/29).
- *Prevention:* Standardize a `lib/lock.sh` helper used by every scheduled script.

**11. Disk becomes full**
- *Root cause:* Unbounded log growth or unrotated temp files from a long-running automation script.
- *Fix:* Log rotation, `mktemp` + `trap cleanup EXIT` guarantees, disk monitoring alerts (Part 38) before it becomes critical.
- *Prevention:* Disk-usage checks as a pre-flight step in any script that writes significant data.

**12. Temporary files are not cleaned up**
- *Root cause:* Script exits via an error path that bypasses manual cleanup code placed only at the bottom of the script.
- *Fix:* `trap cleanup EXIT` (Part 14) — the only reliable pattern.
- *Prevention:* Code review requirement: any `mktemp` usage must be paired with a `trap ... EXIT`.

**13. API returns unexpected JSON**
- *Root cause:* Script assumed a field always exists; `jq` returned `null`, causing a downstream comparison/command to silently misbehave rather than fail loudly.
- *Fix:* `jq -e` (exits non-zero on `null`/false top-level result) and explicit `// empty`/`// "default"` fallbacks.
- *Prevention:* Validate API responses against expected shape before processing; add retries for known-flaky APIs (Part 49).

**14. Command returns non-zero status unexpectedly**
- *Root cause:* A tool's exit-code semantics were misunderstood (e.g., `grep` returns 1 for "no match," which isn't necessarily an "error" in context, but `set -e` treated it as fatal).
- *Fix:* Use `|| true` or restructure as an `if` condition (Part 13's `set -e` pitfalls) when a non-zero exit is an *expected*, handled outcome.
- *Prevention:* Understand and document each external tool's exit-code contract before wrapping it in `set -e` logic.

**15. `set -e` causes unexpected termination**
- *Root cause:* One of the Part 13 pitfalls — command inside an `&&`/`||`/`if` chain, or the last command of a function used in a conditional context.
- *Fix:* Understand exactly when `-e` is suspended (Part 13); test error paths explicitly, not just happy paths.
- *Prevention:* Bats tests specifically targeting failure/edge-case behavior, not just success cases.

**16. Rollback script fails, making things worse**
- *Root cause:* Rollback path itself was never tested — assumed to "just work" the one time it's actually needed.
- *Fix:* Regularly rehearse rollback in staging as part of normal deploy testing.
- *Prevention:* Treat rollback as a first-class, tested code path, not an afterthought.

**17. Secrets leaked into logs**
- *Root cause:* `set -x` tracing was on globally, printing a command line containing a plaintext password/token into a log file that's not access-restricted.
- *Fix:* `set +x` around any secret-handling block (Part 26); never pass secrets as CLI args.
- *Prevention:* Log-scrubbing/redaction tooling; secrets manager integration instead of env-var/plaintext secrets at all.

**18. Zombie/orphaned background processes accumulate**
- *Root cause:* Backgrounded health-check pollers not tracked/killed when the parent script exits or is killed.
- *Fix:* `trap 'kill -- -$$' EXIT` to clean up the whole process group (Part 45).
- *Prevention:* Always track PIDs of backgrounded work and clean up in EXIT traps.

**19. Script behaves differently on different Bash versions**
- *Root cause:* Used a Bash 4.3+ feature (`wait -n`, namerefs) on a Bash 4.1 production host (common on older RHEL/CentOS).
- *Fix:* Check `BASH_VERSION` explicitly for scripts targeting mixed-version fleets, or avoid version-specific features.
- *Prevention:* Pin/document the minimum supported Bash version; test in a container matching production's actual OS/Bash version, not your dev machine's.

**20. Alpine container script fails with "bash: not found"**
- *Root cause:* `#!/bin/bash` shebang in an Alpine-based image without `bash` installed (Part 46).
- *Fix:* `apk add bash`, or rewrite the script to be POSIX `sh`-compatible.
- *Prevention:* Standardize base images, or explicitly document/test the shell available in each image family used.

**21. Health check reports healthy, but the app is actually down**
- *Root cause:* The health check only verified the process/container was running, not that it was actually serving correct responses (Part 38 principle).
- *Fix:* Check an actual HTTP endpoint with expected content/status, not just process existence.
- *Prevention:* Health-check design review as part of onboarding any new service.

**22. Deployment script deploys the wrong version**
- *Root cause:* Script read a cached/stale `git describe` tag because `git fetch --tags` wasn't run first.
- *Fix:* Always `git fetch` before resolving version references in automation.
- *Prevention:* Explicit version-pinning via CLI arg rather than inferring "latest" implicitly wherever possible.

**23. Parallel SSH fleet script overwhelms a rate-limited API/bastion**
- *Root cause:* Unbounded `&`/`wait` parallelism (Part 19) hit 200 hosts simultaneously through a single bastion, exhausting its connection limit.
- *Fix:* Concurrency cap (`wait -n` / GNU `parallel -j N`).
- *Prevention:* Load-test automation against realistic fleet sizes before relying on it at scale.

**24. Script silently does nothing because of an empty glob**
- *Root cause:* `for f in /data/*.csv` where no files matched — Bash left the literal string `*.csv` as the "file," and downstream code tried to process a nonexistent file without checking.
- *Fix:* `shopt -s nullglob` at the top of scripts using globs, or explicit `[[ -e "$f" ]] || continue` checks.
- *Prevention:* Test glob-based scripts against both empty and populated directories.

**25. Config drift because a script fixed a problem "manually" rather than idempotently**
- *Root cause:* An on-call engineer patched a broken config directly on a server during an incident, but the automation script wasn't updated to match, so the next scheduled/CI run reverted the manual fix, reintroducing the outage.
- *Fix:* Immediately encode any manual incident fix back into the automation/IaC source of truth before closing the incident.
- *Prevention:* Treat manual production changes as technical debt that must be reconciled with automation same-day; prefer GitOps-style workflows (K8s/Terraform) precisely because they make drift visible.

---

## PART 52 — Interview Preparation 🏭

Given the volume requested (175 questions total across 5 tiers), here is a strong, representative working set per tier with full answers — the patterns here cover the reasoning you'd need for the rest. Ask me to generate the remaining questions for any specific tier in a follow-up and I'll expand it fully.

### Beginner (sample of 10, representative of the 25)

1. **What is the difference between `sh` and `bash`?** `sh` is the POSIX-standard interface (often `dash`/`ash` under the hood); Bash is a specific, feature-rich shell/superset. Bash-only features (arrays, `[[ ]]`) break under a true `sh`.
2. **What does the shebang line do?** Tells the kernel which interpreter to `exec` the script with; parsed by the OS, not the shell itself.
3. **Difference between `$@` and `$*`?** `"$@"` preserves each argument as a separate word (safe for forwarding args); `"$*"` joins them into one string using `$IFS`.
4. **What does `$?` represent?** The exit status of the last executed command (0 = success).
5. **Difference between single and double quotes?** Single: fully literal. Double: variables and command substitution still expand.
6. **What's the difference between `>` and `>>`?** `>` truncates/overwrites; `>>` appends.
7. **What does `chmod +x script.sh` do, and why is it needed?** Sets the execute permission bit so the file can be run directly (`./script.sh`); without it you must invoke via `bash script.sh`.
8. **What is a variable's scope by default in Bash?** Global, even inside functions, unless declared `local`.
9. **How do you provide a default value for an unset variable?** `${var:-default}` (or `:=` to also persist it).
10. **What's the difference between `[ ]` and `[[ ]]`?** `[[ ]]` is a Bash keyword with safer word-splitting/glob behavior and native pattern/regex matching; `[ ]` is the POSIX `test` command, more portable but less safe with unquoted variables.

### Intermediate (sample of 10, representative of the 30)

1. **Explain `set -euo pipefail` and its limitations.** (Full answer in Part 13 — covers `-e`'s suspension inside conditionals/pipelines-without-pipefail, `-u` catching typos, `pipefail` fixing pipeline exit-status blindness.)
2. **How do you safely loop over files, including ones with spaces?** `find ... -print0 | while IFS= read -r -d '' f; do ... done`, never `for f in $(ls)`.
3. **What's the danger of `eval`?** Re-parses a string as shell code — any embedded shell metacharacters execute, a command-injection vector.
4. **How do you capture both a command's output and exit status?** `output=$(cmd); status=$?` — capture `$?` immediately after, before running anything else.
5. **What does `PIPESTATUS` give you that `$?` doesn't?** The exit status of *every* stage of the last pipeline, not just the final one.
6. **How do you pass an array to a function correctly?** Either a nameref (`local -n ref=$1`, pass the array's name) or rebuild it from `"$@"` after passing `"${arr[@]}"`.
7. **Explain `${var#pattern}` vs `${var%pattern}`.** `#` strips from the front (shortest match), `##` front (longest); `%` strips from the end (shortest), `%%` end (longest).
8. **Why should `local` be used inside functions?** Prevents unintentional global variable clobbering across function calls in larger scripts.
9. **How do you make a script fail if a required environment variable is missing?** `: "${VAR:?VAR must be set}"`.
10. **What's the risk of `rm -rf $dir/*` without quoting/guards?** If `$dir` is empty/unset, expands to `rm -rf /*`; fix with `"${dir:?}"`.

### Advanced (sample of 10, representative of the 40)

1. **Walk through Bash's expansion order.** Brace → tilde → parameter/variable → command substitution → arithmetic → word splitting → pathname/glob expansion (Part 1).
2. **How does `flock` prevent race conditions in cron jobs, and why is it better than a PID file?** Kernel-managed, atomic, auto-released on process exit/crash, unlike a PID file which can go stale if the process dies uncleanly.
3. **Explain the difference between `return`, `exit`, and using `echo` to "return" data from a function.** (Part 12 full answer.)
4. **Why does `set -e` not stop a script when the failing command is inside an `if` condition?** Bash intentionally suspends `-e` in conditional contexts since a non-zero result there is an expected, handled branch, not necessarily fatal.
5. **How do you implement exponential backoff retries in Bash?** Loop with an attempt counter, `sleep "$delay"; ((delay*=2))`, combined with a `timeout` on the underlying command (Part 49).
6. **Explain `${array[@]}` vs `${array[*]}`.** Same distinction as `$@`/`$*` — `[@]` preserves elements; `[*]` joins them into one string.
7. **What's a nameref (`declare -n`), and when would you use one over `eval`?** A safe indirect reference to another variable's name, allowing a function to "return" complex data or operate on a caller-provided variable/array without re-parsing arbitrary strings as code.
8. **How would you debug a script that hangs in production but not locally?** Check for stray `read`s consuming stdin, missing timeouts on network calls, `ssh` without `-n` inside loops, and reproduce under the actual non-interactive/cron-like context (Part 24/51).
9. **Explain how `trap ... EXIT` guarantees cleanup regardless of how a script terminates.** It fires on normal completion, `exit`, and most signal-driven terminations (not `SIGKILL`), making it the most reliable cleanup mechanism versus code at the script's bottom, which is skipped on early error exits.
10. **What does `2>&1 > file` do differently from `> file 2>&1`, and why?** Order matters — redirections apply left to right; `2>&1` first points stderr at the *current* stdout (still the terminal), then stdout is redirected to `file`, leaving stderr on the terminal. `> file 2>&1` redirects stdout to `file` first, then stderr follows stdout to the same file.

### Senior DevOps (sample of 10, representative of the 40)

1. **Design a production deployment script's failure-handling strategy end to end.** (Part 40's Git→Build→Test→Backup→Deploy→Health-check→Rollback pipeline, with `trap`-driven automatic rollback and Slack alerting.)
2. **How do you prevent two instances of a scheduled automation script from running concurrently across a fleet, not just one host?** `flock` handles single-host overlap; for fleet-wide mutual exclusion use a distributed lock (DynamoDB conditional write, Redis `SETNX`, or a leader-election mechanism) rather than local `flock` alone.
3. **How would you make an AWS-resource-creating script idempotent?** Check-then-act (`describe-*` before `create-*`), or use idempotency tokens where the API supports them (e.g., EC2 `ClientToken`).
4. **What's your strategy for secrets in Bash automation across CI/CD?** Never hardcode; pull from a secrets manager/vault at runtime; use short-lived credentials (OIDC federation) over long-lived static keys; never pass secrets as CLI args (`ps aux` exposure).
5. **How do you validate a deployment before promoting it to 100% traffic?** Canary + soak period + automated metric-based go/no-go decision (Part 40), not manual eyeballing.
6. **Explain your approach to testing infrastructure automation scripts.** Bats unit tests for library functions with mocked external commands, ShellCheck static analysis in CI, and staged rehearsal of destructive/rollback paths in a non-prod environment before trusting them in prod.
7. **How do you handle a script that must run identically across Ubuntu, Alpine, and macOS dev machines?** Either constrain to POSIX `sh` syntax, or explicitly ensure Bash 4+ is present everywhere (`apk add bash`), and test in containers matching each target, not just locally.
8. **Describe a real production incident caused by Bash and how you'd prevent recurrence.** (Draw from Part 51 — pick one, walk the full symptoms→investigation→fix→prevention chain.)
9. **How do you structure a growing collection of ops scripts so they don't become unmaintainable?** `lib/`+`bin/` separation (Part 41), shared logging/error/validation libraries, ShellCheck+Bats in CI, and knowing when a script has outgrown Bash entirely (Part 43).
10. **How do you safely roll out a change to a cron-based automation fleet without an outage window?** Idempotent, backward-compatible scripts; deploy the new script version alongside the old with a feature flag/env var before fully cutting over; monitor the first few scheduled runs closely.

### Senior Platform Engineer (sample of 10, representative of the 40)

1. **When would you choose Bash vs. Ansible vs. Terraform for a given automation task, and why?** (Part 43 table — Bash for gluing existing CLIs/orchestration; Ansible for idempotent fleet configuration; Terraform for declarative, stateful infrastructure provisioning.)
2. **How do you design self-service tooling (Bash CLIs) for other engineers on your platform team?** Clear `--help`, strict input validation with helpful error messages, dry-run modes, consistent subcommand structure (Part 23/41), and treating the CLI itself as a product with tests and versioning.
3. **How do you prevent "worked in staging, broke in production" for Bash-driven deployment tooling?** Ensure staging mirrors production's OS/Bash version/permissions/network topology exactly; rehearse rollback in staging, not just forward deploys.
4. **How would you build observability into a fleet of Bash automation scripts?** Structured logging (timestamped, to a central log aggregator), meaningful exit codes wired into monitoring (Part 38), and failure alerts routed through the same on-call system as application incidents — automation failures are production incidents.
5. **What's your philosophy on "when has this Bash script grown too large"?** When it needs real data structures beyond simple arrays/associative arrays, complex branching logic beyond a few `case`/`if` levels, or shared use across teams requiring proper packaging/versioning/testing infrastructure — that's the signal to port to Python/Go.
6. **How do you handle multi-region/multi-account AWS automation safely from Bash?** Explicit `aws sts get-caller-identity`/account-ID checks before any destructive action, per-environment credential isolation (never a single script silently able to touch prod and staging via ambient credentials), and dry-run defaults.
7. **Describe how you'd build a platform-wide "golden path" for writing new ops scripts.** A shared `lib/` template repo (logging, error handling, validation, retry, lock helpers from this course), a linting/testing CI template, and a documented checklist (Part 47) enforced via PR review or a pre-commit hook.
8. **How do you reason about the blast radius of a Bash automation bug?** Consider what credentials/scope the script runs with (least privilege), whether it's idempotent (limits damage from double-runs), whether it has dry-run/guard-rails for destructive actions, and whether a single script instance can affect multiple environments simultaneously (it shouldn't).
9. **How would you migrate a legacy fleet of ungoverned shell scripts into a maintainable platform?** Inventory and ShellCheck everything first (find "loaded gun" scripts using `eval`/unguarded `rm`), triage by blast radius, refactor into shared libraries incrementally, and add tests/CI gates before further changes — don't rewrite everything at once.
10. **What's a circuit-breaker pattern, and would you implement one in Bash or elsewhere?** (Part 49 — concept explained; in practice, real circuit breakers belong at the service-mesh/application layer, but the retry/backoff discipline that underlies them absolutely belongs in your Bash automation.)

*(For the full 175-question bank across all five tiers with every answer written out, tell me and I'll generate the remaining questions in a follow-up pass — this sample demonstrates the reasoning depth expected at each level.)*

---

## PART 53 — Bash Cheat Sheet

```
VARIABLES               ${var:-default} ${var:=default} ${var:+val} ${var:?err}
SPECIAL VARS            $0 $1..$9 $# $@ $* $? $$ $! $- BASH_SOURCE FUNCNAME PIPESTATUS
STRINGS                 ${#s} ${s:off:len} ${s#p} ${s##p} ${s%p} ${s%%p} ${s/a/b} ${s//a/b} ${s^^} ${s,,}
CONDITIONS              [[ -z ]] [[ -n ]] [[ -e/-f/-d/-r/-w/-x/-s ]] [[ a == b ]] [[ n -eq m ]] (( expr ))
LOOPS                   for x in ...; do..done | while cond; do..done | for ((i=0;i<n;i++))
FUNCTIONS               name() { local x="$1"; ...; return N; }
ARRAYS                  arr=(a b c) / declare -A m=([k]=v) / "${arr[@]}" / "${!m[@]}" / "${#arr[@]}"
REDIRECTION             > >> < 2> 2>> 2>&1 &> /dev/null
PIPES                   cmd1 | cmd2 (set -o pipefail; check PIPESTATUS)
PROCESSES               cmd & ; $! ; wait ; jobs ; kill -TERM/-9 pid ; pgrep -f ; pkill -f
SIGNALS/TRAPS           trap 'cmd' EXIT|ERR|INT|TERM ; flock -n 200
ERROR HANDLING          set -euo pipefail ; trap 'log_error "line $LINENO"' ERR ; die() { ...; exit 1; }
REGEX                   grep -E 'a|b' ; [[ $s =~ ^regex$ ]] ; ${BASH_REMATCH[1]}
GREP/SED/AWK            grep -E/-v/-c/-o ; sed 's/a/b/g' -i.bak ; awk -F, '{print $1}'
JQ                      jq -r '.field' ; jq -n --arg k v '{k:$k}' ; jq -e '.x // empty'
CURL                    curl -sf -o out -w "%{http_code}" --connect-timeout 5 --max-time 30 url
SSH                     ssh -o BatchMode=yes -o ConnectTimeout=10 -o StrictHostKeyChecking=accept-new host cmd
GIT                     git rev-parse --short HEAD ; git status --porcelain ; git describe --tags --abbrev=0
DOCKER                  docker inspect --format='{{.State.Health.Status}}' c ; docker image prune -af
KUBERNETES              kubectl rollout status/undo ; kubectl get -o json | jq ; kubectl apply -f (idempotent!)
AWS CLI                 aws sts get-caller-identity ; aws ... --query '...' --output json|text|table
DEBUGGING               bash -x / bash -n ; set -x/+x ; PS4='+ ${BASH_SOURCE}:${LINENO}: ' ; shellcheck file.sh
ARG PARSING             getopts ":e:v:h" opt ; shift $((OPTIND-1)) ; manual while-case for --long-opts
IDEMPOTENCY             check-then-act ; mkdir -p ; kubectl apply ; grep -qxF || append
RETRIES                 retry() { until "$@"; do ...; sleep $d; ((d*=2)); done }  — always pair with `timeout`
```

---

## PART 54 — 30-Day Learning Plan (2 hrs/day)

### Week 1 — Fundamentals & Language Core
- **D1:** Part 1 (execution model, PATH, startup files) + set up a scratch VM/container. Exercise: diagnose a fake "PATH under cron" issue.
- **D2:** Part 2 (variables, parameter expansion `:-`/`:=`/`:+`/`:?`). Exercise: build an env-var validation block.
- **D3:** Part 3 (special variables, `$@` vs `$*`, `PIPESTATUS`). Exercise: arg-forwarding wrapper function.
- **D4:** Part 4 (I/O, file descriptors, redirection order). Exercise: split stdout/stderr logging script.
- **D5:** Part 5 + 6 (command substitution, quoting). Exercise: find & fix quoting bugs in a provided broken script.
- **D6:** Part 7 (conditionals, `[ ]` vs `[[ ]]`). Exercise: validation block for a deploy script.
- **D7:** Review + ShellCheck every script from the week; fix all warnings.

### Week 2 — Control Flow, Data, Text
- **D8:** Part 8 (loops, safe file iteration). Exercise: safe recursive file processor with `find -print0`.
- **D9:** Part 9 (case statements). Exercise: service-management dispatcher script.
- **D10:** Part 10 (arrays, associative arrays). Exercise: env→AWS-account-ID map + validator.
- **D11:** Part 11 (string manipulation). Exercise: Docker image reference parser, slugify function.
- **D12:** Part 12 (functions, return vs echo vs exit). Exercise: build `lib/validation.sh`.
- **D13:** Part 16 (grep/sed/awk). Exercise: Nginx log analyzer (top IPs, status breakdown).
- **D14:** Part 17 (regex) + review. Exercise: extract IPs/URLs/status codes from logs with regex.

### Week 3 — Advanced Bash, Robustness, Integrations
- **D15:** Part 13 (exit codes, `set -euo pipefail` deep dive + pitfalls). Exercise: demonstrate a `set -e` blind spot.
- **D16:** Part 14 (traps). Exercise: backup script with guaranteed `mktemp` cleanup.
- **D17:** Part 18 + 19 (processes, parallelism, `flock`). Exercise: parallel SSH fact-gatherer with concurrency cap.
- **D18:** Part 23 (arg parsing — `getopts` + long flags). Exercise: full `deploy.sh --environment --version` CLI.
- **D19:** Part 24 + 25 (debugging, ShellCheck). Exercise: run ShellCheck across everything built so far; fix all.
- **D20:** Part 26 (security). Exercise: audit and fix injection/quoting/secrets risks in earlier scripts.
- **D21:** Part 27 + 35 (SSH automation, APIs/curl/jq). Exercise: retry-wrapped API health checker.

### Week 4 — Production Automation & Real Projects
- **D22:** Part 31 (Docker automation). Exercise: build → smoke-test → push → deploy script.
- **D23:** Part 32 (Kubernetes automation). Exercise: rollout + automatic rollback + health-check trio.
- **D24:** Part 33 (AWS automation). Exercise: EC2 inventory + S3 backup scripts.
- **D25:** Part 36 + 37 (JSON/YAML, log automation). Exercise: paginated API aggregator with `jq`.
- **D26:** Part 38 + 39 (monitoring, backup). Exercise: multi-check `healthcheck.sh` dispatcher; full backup script with integrity verification.
- **D27:** Part 40 + 48 + 49 (deployment pipeline, idempotency, retries). Exercise: full Git→Build→Test→Deploy→Health-check→Rollback pipeline.
- **D28:** Part 41 + 42 (architecture, Bats testing). Exercise: refactor all prior scripts into a `lib/`+`bin/` structure with Bats tests.
- **D29:** Part 47 (production checklist) + Part 51 (incidents). Exercise: apply the full checklist to your capstone project; write up 3 incident postmortems in the S-I-C-R-F-P format for bugs you personally hit this month.
- **D30:** Capstone: build Project 19 (Complete DevOps Automation CLI) end-to-end — deploy/rollback/health-check/backup subcommands, tested, ShellChecked, documented.

---

## PART 55 — Final Skill Matrix

| Skill | Beginner | Intermediate | Advanced | Senior DevOps | Platform Eng | Expert |
|---|---|---|---|---|---|---|
| Bash fundamentals | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Variables & expansion | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Conditions | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Loops | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Functions | — | ✅ | ✅ | ✅ | ✅ | ✅ |
| Arrays | — | ✅ | ✅ | ✅ | ✅ | ✅ |
| Text processing (grep/sed/awk) | — | ✅ | ✅ | ✅ | ✅ | ✅ |
| Regex | — | — | ✅ | ✅ | ✅ | ✅ |
| File processing / find+xargs | — | ✅ | ✅ | ✅ | ✅ | ✅ |
| Processes & job control | — | — | ✅ | ✅ | ✅ | ✅ |
| Signals & traps | — | — | ✅ | ✅ | ✅ | ✅ |
| Error handling (`set -euo pipefail`) | — | ✅ | ✅ | ✅ | ✅ | ✅ |
| Debugging (`-x`, PS4) | — | ✅ | ✅ | ✅ | ✅ | ✅ |
| Security (quoting, no `eval`, secrets) | — | ✅ | ✅ | ✅ | ✅ | ✅ |
| ShellCheck | — | ✅ | ✅ | ✅ | ✅ | ✅ |
| Testing (Bats) | — | — | ✅ | ✅ | ✅ | ✅ |
| Networking basics for scripting | — | ✅ | ✅ | ✅ | ✅ | ✅ |
| APIs / curl / jq | — | ✅ | ✅ | ✅ | ✅ | ✅ |
| SSH automation | — | ✅ | ✅ | ✅ | ✅ | ✅ |
| Docker automation | — | ✅ | ✅ | ✅ | ✅ | ✅ |
| Kubernetes automation | — | — | ✅ | ✅ | ✅ | ✅ |
| AWS automation | — | — | ✅ | ✅ | ✅ | ✅ |
| CI/CD integration | — | ✅ | ✅ | ✅ | ✅ | ✅ |
| Monitoring / health checks | — | ✅ | ✅ | ✅ | ✅ | ✅ |
| Production automation (deploy/rollback) | — | — | ✅ | ✅ | ✅ | ✅ |
| Idempotency & resilience design | — | — | — | ✅ | ✅ | ✅ |
| Performance (know when to leave Bash) | — | — | ✅ | ✅ | ✅ | ✅ |
| Architecture (lib/bin, multi-script systems) | — | — | — | ✅ | ✅ | ✅ |
| Fleet/multi-cloud orchestration | — | — | — | — | ✅ | ✅ |
| Mentoring / golden-path tooling design | — | — | — | — | ✅ | ✅ |

**You'll know you've reached "Expert / Platform Engineer" level when:** you can read an unfamiliar 500-line production Bash script cold and immediately spot its quoting bugs, missing error handling, and idempotency gaps; you instinctively reach for `flock`, `trap`, and `set -euo pipefail` without thinking; and you know exactly when to stop writing Bash and reach for Python, Ansible, or Terraform instead.
