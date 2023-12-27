# beltaloada: an out of this world shell build system

> The more you share, the more your bowl will be plentiful.

—Belter asteroid miner saying, *The Expanse*

# ABOUT

beltaloada is an unconventional build system. It is written by and for shell script users.

It is not a tool you can install. It is a set of guidelines. beltaloada cannot fail. beltaloada is a way of living.

# EXAMPLE

* [example/build](example/build)

# MOTIVATION

sh deserves a better build system. Unlike C and C++, there is no widely accepted build system for managing sh development tasks. beltaloada describes such a system.

## Why not make?

Notably, `make` may offer deduplication and other performance gains, in case the user ends up requesting multiple, redundant task subtrees. Also, `make` tends to cache temporary files well for project with many, inter-task artifacts. However, `make` is usually outclassed by modern programming language build systems.

In the context of shell projects, `make` encourages waste. The `.PHONY` target scales linearly (poorly) with the number of build tasks for projects that do not generate many file artifacts, such as shell projects.

`make` is difficult to statically analyze. There are few `make` linters, and even fewer linters able to lint shell commands embedded in makefiles. This makes `make` doubly fragile, inviting avoidable bugs into the build system.

`make` presents a (minor) attack surface, for projects concerned with the size of the development environment. sh is Turing complete, so it stands to reason that we should not need to install a tool written in C to manage sh code. We generally prefer installing as few development components as possible, so that the build environment is clean and quick to setup on new machines. The alternative requires additional linting for the makefile, which means more tools, more programming languages... ballooning the development environment just to support some shell scripts.

`make` represents a language completely apart from `sh`. Accordingly, this significantly increases the cognitive load when debugging projects involving `sh` + `make`. For instance, `makefiles` that neglect to properly backslash (`\`) piped multiline commands can lead to accidents, due to how the make syntax interacts with shell syntax. This is less of a problem when operating entirely within a single language, shell.

When diagnosing build problems, it's faster to sift through a single body of documentation rather than two bodies of documentation. Hence why it is pragmatic to use *Mage* to configure Go build tasks, *Shake* to configure Haskell build tasks, and so on. Implementing the build system in the same language as the main project application or library accelerates debugging and maintenance.

## Why not (insert bash build system here)?

(GNU) bash is not the only game in town. There are dozens of `sh` implementations, from ash to zsh. For instance, macOS currently defaults to zsh, while WSL defaults to bash, while Alpine Linux defaults to ash. Even a small team of engineers are immediately confronted with conflicting sh implementations, and likely experience errors when running scripts on multiple nodes. In the cloud, Works on My Machine^TM is not a valid answer.

We favor no particular sh implementation over the other. In fact, implementations come with their own laundry list of tradeoffs, and often vary between local development environments, optimized Docker containers, and end user environments. The way to manage the madness of fragmentation involves targeting the lowest common demoninator, POSIX sh.

To be clear, Command Prompt, fish, ion, PowerShell, rc, (t)csh, Thompson sh, and other non POSIX conforming shells are not directly supported by the beltaloada code snippets below. However, the concepts we explore will very likely have equivalent syntax in fringe shells. In that case, consult your shell's documentation.

# REQUIREMENTS

* a UNIX environment with [coreutils](https://www.gnu.org/software/coreutils/) / [base](http://ftp.freebsd.org/pub/FreeBSD/releases/) / [macOS](https://www.apple.com/macos) / [WSL](https://learn.microsoft.com/en-us/windows/wsl/install) / etc.

## Recommended

* [awk](https://pubs.opengroup.org/onlinepubs/9699919799/utilities/awk.html)

# BELTALOADA GUIDELINES

## Nest executable and importable project shell scripts in bin/ and lib/

Modern programming languages have established conventions for project directory trees: A directory for executables, a directory for importable libraries, and a directory for other project files.

For shell projects, these directories align to `bin`, `lib`, and `.` respectively. Put any executable scripts in the first, any importable library scripts in the second, and the build system script in the third. Doing so greatly clarifies the structure of the project, which otherwise could vary wildly. For instance, housing executable shell applications in `bin` simplifies the installation projects to merely adding the `bin` directory to `PATH`. A little structure accelerates a ton of guesswork! It also reduces the need for documentation and manual steps.

## Lint build scripts

Remember to include the sh build system configuration script itself, along with the rest of your shell code. If you don't lint the build system, then you might as well slap on `make`. And assume all the risks that come with that.

## Declare tasks in a `build` file

Place a `build` text file at the **top level directory** of the project.

No file extension.

LF line endings, with UTF-8/ASCII encoding.

Use a simple POSIX style shebang, like:

```sh
#!/bin/sh
```

Minimize use of bashisms and other interpreter specific syntax in the build script. Minimize use of GNU and BSD specific commands and flags.

Ensure the current working directory matches the top level project directory.

Add executable permissions:

```console
$ chmod +x build
```

Run the build:

```console
$ ./build
```

For the same reason, avoid housing executables at the top level directory of your projects. Generated executables are commonly nested in a temporary directory like `bin`, `target`, `dist`, `.node_modules`, etc., and even excluded from version control. Permanent, general purpose scripting language executables are commonly nested as source code in a permanent directory like `lib`. Nonexecutable, importable shell libraries may be nested in a directory like `lib`. Permanent, executable shell scripts may be nested in a directory like `bin`. Put together, this sounds rather complicated. But even a single degree of directory nesting is often sufficient to reduce

Avoid adding the top level project directory to `PATH`. Doing so risks resolving the wrong `build` script from different projects.

Compiled programming languages tend to reserve a temporary directory like `bin`, `target`, `dist`, etc. as a safe location for binary executables. General purpose scripting languages reserve a temporary directories (`.node_modules`,

For general purpose programming languages, we have `bin` (gitignorable in C, C++, etc.), `target` (Java), `dist` (Haskell), `~/go/bin` (Go), `.node_modules` (Node.js)

Instead, house any permanent executable files in a subdirectory like `lib`, `bin`, etc.

## Unset IFS and enable sh safety flags

After the shebang, unset the IFS variable and enable sh safety flags. These work to catch many kinds of common coding mistakes that may occur in the build script.

```sh
unset IFS
set -euf
```

## Initialize a `DEFAULT_TASK` variable

```sh
DEFAULT_TASK=all
```

This resembles the `all` conventional task at the top of traditional makefiles.

## Implement task hierarchies as shell functions

```sh
DEFAULT_TASK=all

all() {
    lint
    test
}

lint() {
    shellcheck ./build lib/*.sh
}

test() {
    test_messaging
    test_noise
}

test_messaging() {
    test_console
    test_stream
}

test_console() {
    echo "Howdy"
}

test_stream() {
    echo "zebra\naardvark" |
        sort
}

test_noise() {
    mplayer fixtures/static.mp3
}
```

Above, we have an example hierarchy.

This hierarchy is configured with a root level `all` task. The `all` task is set as the default, when no explicit tasks are supplied to the `./build` command. The `all` task triggers some task subtrees, in this case, `lint` and `test`. The lint task demonstrates a ShellCheck lint for the build script and some project `*.sh` scripts in a `lib` directory. Leaves at the ends of the task tree, contain the ordinary shell commands that fulfil their duties.

The names and depth of your task trees will vary from project to project. Recommend grouping and ordering tasks, for example logically, alphabetically, leftmost tree traversal order. Whenever execution order is otherwise arbitrary.

Warning, when the command(s) implementing a task begin with the same name as the task, then explicitly prefix those steps with `command`:

```sh
safety() {
    command safety check
}
```

This mitigates recursion issues.

## Clean up extraneous resources with a `clean` task

For instance, a project that builds tarball artifacts files during builds:

```sh
clean() {
    rm -f *.tgz
}
```

## Declare a `help` task to enumerate configured tasks

And now for the hard part. Generating a help menu from a shell script, using only POSIX features.

```sh
TASK_NAMES="$(
    awk '/^[a-zA-Z0-9_]+\(\)/ {
        split($0, a, /\(\)/);
        print a[1]
    }' "$0" |
        sort
)"

for TASK_NAME in $TASK_NAMES; do
    eval "BELTALOADA_TASK_${TASK_NAME}='1'"
done

help() {
    echo 'Usage: ./build [<task> [<task> [<task>...]]]

Tasks:
'

    for TASK in $TASK_NAMES; do
        echo "* $TASK"
    done
}
```

Useful for quickly looking up task names, without having to read the build system source code. As a side effect, internal variables with the dynamic binding pattern `BELTALOADA_TASK_<task>` are set as entries in a logical hashmap, for the sake of the entrypoint's validation needs.

Shell languages are unfortunately bereft of basic data structures for fast, reliable programming. Bash may provide more (associative) array functionality, though the primary hashmap available to POSIX sh is the file system, and file system storage is slower than computation.

`read` may offer a safer alternative to `eval` for writing variables with dynamic names. For the sake of parity, we use `eval` for both insert and lookup.

## Implement an entrypoint to trigger named tasks

```sh
if [ "$#" -eq 0 ]; then
    "$DEFAULT_TASK"
    exit 0
fi

for SUPPLIED_TASK in "$@"; do
    KEY="BELTALOADA_TASK_${SUPPLIED_TASK}"
    VAL="$(eval "echo \"\${$KEY-0}\"")"

    if [ "$VAL" = '0' ]; then
        help
        exit 1
    fi

    "$SUPPLIED_TASK"
done
```

This enables the build script to run the default task when no arguments are supplied to `./build`, and run any named tasks when supplied as `./build <task> [<task> [<task> ...]]]`.

This example uses external tools, POSIX `awk` and `sort`, to extract the task names and order them in a pleasing fashion for the console. On various UNIX systems, these tools are commonly provided in the `base`, `coreutils`, and/or `awk` packages. They may or may not be automatically installed in an environment, depending on the minimalism of the operating system.

Together, the DRY principle and input validation raise the quality bar for our build system. This is critical, because any faults in a cloud CI/CD build are often obfuscated by subtle shell scripting bugs.

## Move complicated processes to `.beltaloada/{bin,lib}/`

A build task may require more intricate work, such as UNIX `find` queries. Try to isolate the reusable, generalized aspects of these steps into useful executables or libraries. Execute them from from `.beltaloada/bin/<script>`, or import them with `. .beltaloada/lib/<script>` from the main `build` script. This keeps the main `build` script more maintainable.

Users may copy these helper scripts between different beltaloada projects. A middle ground between raw copypasta and version pinned dependencies, in a shell world lacking an official POSIX sh package manager.

Even publish them as new projects in their own right.

## Declare provisioning commands in an `install` file

Automate development environment steps with an executable `install` script in the top level project directory. For instance:

```sh
pip3 install --upgrade pip setuptools
pip3 install -r requirements-dev.txt
```

For some popular auxiliary shell project development tools.

## Reference build or install command(s) in your documentation

Let's demystify software development.

Cite high level `./build` or `./install` commands in a documentation file like `README.md` or `DEVELOPMENT.md`. That way, contributors who may be unfamiliar with the build system can learn how to build the project more quickly. Better yet, link from your project documentation to the beltaloada project.

New contributors should be informed how to compiled a compiled project, how to test a project with unit/integration tests, how to lint a project for coding quirks, and how to audit for security vulnerabilities. The exact low level details are less important than the short command line `./build` and `./install` steps to accomplish these high level goals.

# SEE ALSO

* Inspiration from [nobuild](https://github.com/tsoding/nobuild), a convention for C/C++ build systems
* [bashate](https://github.com/openstack/bashate), a shell script style linter
* [Gradle](https://gradle.org/), a build system for JVM projects
* [lake](https://luarocks.org/modules/steved/lake), a Lua task runner
* [Mage](https://magefile.org/), a task runner for Go projects
* [npm](https://www.npmjs.com/), [Grunt](https://gruntjs.com/), Node.js task runners
* [POSIX make](https://pubs.opengroup.org/onlinepubs/009695299/utilities/make.html), a task runner standard for C/C++ and various other software projects
* [Rake](https://ruby.github.io/rake/), a task runner for Ruby projects
* [Shake](https://shakebuild.com/), a task runner for Haskell projects
* [ShellCheck](https://www.shellcheck.net/), a shell script linter with a rich collection of rules for promoting safer scripting
* [slick](https://github.com/mcandre/slick), a linter to enforce stricter, unextended POSIX sh syntax compliance
* [stank](https://github.com/mcandre/stank), a collection of POSIX-y shell script linters
* [tinyrick](https://github.com/mcandre/tinyrick) for Rust projects

🚀