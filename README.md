# Lake

[![Circle CI](https://circleci.com/gh/takagi/lake.svg?style=shield)](https://circleci.com/gh/takagi/lake)
[![Coverage Status](https://coveralls.io/repos/takagi/lake/badge.svg?branch=master&service=github)](https://coveralls.io/github/takagi/lake?branch=master)

Lake is a GNU make or Ruby's [rake](https://github.com/ruby/rake) like build utility in Common Lisp. Instead of Makefile or Rakefile, it uses Lakefile where tasks executed are defined.

Make is, originally, a program to build executable files to compile multiple source files and resolve their dependency, however its use case may not be limited as a build utility but shell command management is also good. Lake's main use case would be the latter.

## Usage

In lake, you use `Lakefile` instead of `Makefile` or `Rakefile`.

    (use-package '(:lake :cl-syntax))
    (use-syntax :interpol)

    ;; Tasks that build an executable with dependency.
    (defparameter cc (or (ros:getenv "CC") "gcc"))

    (file "hello" ("hello.o" "message.o")
      (sh #?"${cc} -o hello hello.o message.o"))
    
    (file "hello.o" ("hello.c")
      (sh #?"${cc} -c hello.c"))
    
    (file "message.o" ("message.c")
      (sh #?"${cc} -c message.c"))
    
    (task "clean" ()
      (sh "rm -rf hello hello.o message.o"))

From REPL, you can call `hello` task with the following form. Ensure that the REPL process' current directory is same as the `Lakefile`.

    (lake:lake :target "hello")

Or you can also call it from command line, which would be the main use case of lake.

    $ lake hello

As a tip, you can generate an empty Lakefile with package related boilerplates in the current directory as:

    $ lake-tools init

Further detail, please see [Lakefile](#Lakefile) section and [API](#API) section.

## Install

**Since lake is not on Quicklisp yet, please use its local-projects feature for now.**

    $ cd ~/.roswell/local-projects
    $ git clone git@github.com:takagi/lake.git

    $ cd lake
    $ ros -l lake.asd install lake    # install LAKE command

You can install lake via Quicklisp,

    (ql:quickload :lake)

or you can also install it using Roswell including `lake` command. Ensure that `PATH` environment variable contains `~/.roswell/bin`.

    $ ros install lake
    $ which lake
    /path/to/home/.roswell/bin/lake

## Lakefile

Lake provides the following forms to define tasks and namespaces in `Lakefile`:
- **Task** fundamental concept processing a sequence of shell commands
- **File Task** a task resolving file dependency with up-to-date check
- **Directory Task** a task that ensures a directory exists.
- **Namespace** grouping up multiple tasks to magage them.

### Task

    TASK task-name dependency-list form*

`task` represents a sequence of operations to accomplish some task. `task-name` is a string that specifies the target task by its name. `dependency-list` is a list of tasks names on which the target task depends. The dependency task names are given in both relative and absolute manner, which begins with a colon `:`. `form`s can be any Common Lisp forms.

    $ cat Lakefile
    ...
    (namespace "hello"

      (task "foo" ("bar")               ; dependency task in relative manner
        (echo "hello.foo"))

      (task "bar" (":hello:baz")        ; dependency task in absolute manner
        (echo "hello.bar"))

      (task "baz" ()
        (echo "hello.baz")))

    $ lake hello:foo
    hello.foo
    hello.bar
    hello.baz

### File

    FILE file-name dependency-list form*

`file` task represents a sequence of operations as well as `task` except that it is executed only when the target file is out of date. `file-name` is a string that specifies the target file's name. `dependency-list` is a list of tasks or file names on which the target file depends. The dependency task/file names are given in both relative and absolute manner, which begins with a colon `:`. `form`s can be any Common Lisp forms.

    $ cat Lakefile
    ...
    (file "hello" ("hello.c")
      (sh "gcc -o hello hello.c"))

    $ lake hello
    $ ls
    Lakefile hello hello.c

### Directory

    DIRECTORY directory-name

`directory` task represents a task that ensures a directory with name of `directory-name` exists. `directory` task does not depend on other tasks.

    $ cat Lakefile
    ...
    (directory "dir")

    $ lake dir
    $ ls
    Lakefile dir

### Namespace

    NAMESPACE namespace form*

`namespace` groups up multiple tasks for ease of their management. `namespace` is a string that specifies the namespace. Tasks in a namespace are refered with the namespace as prefix separated with a colon `:`. Namespaces may be recursive.

    $ cat Lake
    ...
    (namespace "foo"
      (namespace "bar"
        (task "baz" ()
          (echo "foo.bar.baz")))))

    $ lake foo:bar:baz
    foo.bar.baz

## API

### [Function] lake

    LAKE &key target pathname parallel verbose

Loads a Lakefile specified with `pathname` to execute a task of name `target` defined in the Lakefile. Not nil `parallel` enables parallel task execution. Not nil `verbose` provides verbose mode. If `target` is not given, `"default"` is used for the default task name. If `pathname` is not given, a file of name `Lakefile` in the current directory is searched for. You should be aware that where the Common Lisp process' current directory is.

    (lake :target "hello")

### [Function] sh

    SH command &key echo

Spawns a subprocess that runs the specified `command` given as a string. When `echo` is `t`, prints `command` to the standard output before runs it. Actually it is a very thin wrapper of `uiop:run-program` provided for UNIX terminology convenience. 
Acompanied with cl-interpol's `#?` reader macro, you get more analogous expressions to shell scripts.

    (defparameter cc "gcc")

    (task "hello" ("hello.c")
      (sh #?"${cc} -o hello hello.c"))

### [Function] echo

    ECHO string

Writes the given `string` into the standard output followed by a new line, provided for UNIX terminology convenience.

    (task "say-hello" ()
      (echo "Hello world!"))

### [Function] execute

    EXECUTE target

Executes a task specified with `target` as a string within another. The name of the target task is resolved as well as `task` macro's dependency list in both relative and absolute manner.

    (task "hello" ("hello.c")
      (sh "gcc -o hello hello.c")
      (execute "ls"))

    (task "ls" ()
      (sh "ls -l"))

## Command Line Interface

### lake

Lake provides its command line interface as a roswell script.

    SYNOPSIS

        lake [ -f lakefile ] [ options ] ... [ targets ] ...

    OPTIONS

        -f FILE
            Use FILE as a Lakefile.
        -h
            Print usage.
        -j
            Execute multiple tasks in parallel.
        -v
            Verbose mode.

    EXAMPLE

        $ lake hello:foo hello:bar

### lake-tools

Lake also provides `lake-tools` command as a roswell script which is a complementary program to provide some useful goodies.

Here shows `lake-tools` command's detail. `lake-tools init` command would be replaced by roswell's `ros init` facility (See issue #12).

    SYNOPSIS

        lake-tools COMMAND

    COMMANDS

        init    Create an empty Lakefile with boilerplates in current directory.

    EXAMPLE

        $ lake-tools init
        $ ls
        Lakefile

## Brief History

Originally lake was called clake, and [@Rudolph-Miller](https://github.com/Rudolph-Miller) gave its name and concept on his GitHub repository. Then, [@takagi](https://github.com/takagi) forked it to design, implement, test, document and CI independently. Afterwards it is renamed to lake.

## Author

* Rudolph Miller (chopsticks.tk.ppfm@gmail.com)
* Masayuki Takagi (kamonama@gmial.com)

## Copyright

Copyright (c) 2015 Rudolph Miller (chopsticks.tk.ppfm@gmail.com)

## License

Licensed under the MIT License.
