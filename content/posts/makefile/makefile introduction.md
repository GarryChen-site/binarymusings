---
title: "Makefile Tutorials Part 1 : Basic make Rules, Targets & Clean"

description: "Learn Makefile basics for beginners: how make works, defining rules, targets, dependencies, variables, clean builds, and included makefiles for C projects."

summary: "This first Makefile tutorial walks through how make works using a small C project example. You’ll learn the core rule syntax—targets, prerequisites, and recipes—plus how to use variables, implicit rules, clean targets, and included makefiles to get fast, incremental builds and a clean project structure."

date: 2025-11-13
series: ["Makefile Tutorial"]
weight: 1
tags: ["makefile", "build-automation", "c-language", "gnu-make", "tutorial"]
author: ["Garry Chen"]
cover:
  image: 
  hiddenInList: 
  caption: ""

---




This is the first article in my **Makefile tutorials** series. We’ll start with a practical introduction to how `make` works, using a small C project to learn basic rules, targets, dependencies, variables, and the classic `clean` target.

When the make command is executed, it requires a makefile to tell it how to compile and link the program.

First, let’s use an example to explain the writing rules of a makefile, so that you can get an intuitive understanding. This example is taken from the GNU make manual. In this example, our project has 8 C files and 3 header files, and we need to write a makefile to tell the make command how to compile and link these files. Our rules are as follows:
1. If the project has never been compiled before, all C files must be compiled and linked.
2. If some of the C files have been modified, only the modified C files should be recompiled, and the target program relinked.
3. If a header file has been changed, then all C files that include this header file should be recompiled, and the target program relinked.

As long as our makefile is well-written, all of the above can be completed with a single make command. The make command will automatically and intelligently determine which files need to be recompiled based on the current file modification state, and then automatically compile and link accordingly.


## The rules of makefile

Before explaining this makefile, let’s first take a brief look at the structure of a makefile rule.

``` shell
target ... : prerequisites ...
    recipe
    ...
    ...
```
### target
Can be an object file, an executable file, or a label. 
### prerequisites
The files and/or targets that the target depends on.
### recipe
The commands to execute for this target (any shell commands).

This describes a file dependency relationship, meaning that one or more target files depend on the files listed in the prerequisites. The commands defined in the recipe describe how to generate them.
In simpler terms: 
> if any file in the prerequisites list is newer than the target file, the commands in the recipe will be executed.

This is the core of makefiles — their fundamental concept.

## Example project for this Makefile tutorial

As mentioned earlier, if a project has 3 header files and 8 C files, then to fulfill the three rules above, our makefile should look like this:

``` makefile
edit : main.o kbd.o command.o display.o \
        insert.o search.o files.o utils.o
    cc -o edit main.o kbd.o command.o display.o \
        insert.o search.o files.o utils.o

main.o : main.c defs.h
    cc -c main.c
kbd.o : kbd.c defs.h command.h
    cc -c kbd.c
command.o : command.c defs.h command.h
    cc -c command.c
display.o : display.c defs.h buffer.h
    cc -c display.c
insert.o : insert.c defs.h buffer.h
    cc -c insert.c
search.o : search.c defs.h buffer.h
    cc -c search.c
files.o : files.c defs.h buffer.h command.h
    cc -c files.c
utils.o : utils.c defs.h
    cc -c utils.c
clean :
    rm edit main.o kbd.o command.o display.o \
        insert.o search.o files.o utils.o
```

The backslash (\) represents a line continuation, which makes the makefile easier to read. You can save this content in a file named "makefile" or "Makefile", then simply run `make` in the directory to generate the executable edit. If you want to delete the executable file and all intermediate object files, just run `make clean`.

In this makefile, targets include the executable edit and the intermediate object files (`*.o`). Prerequisites are the `.c` and `.h` files listed after the colons. Each `.o` file has its own dependency list, and all of those `.o` files are dependencies of the edit target. Essentially, dependencies describe which files are used to generate the target, or in other words, which files were updated to create the target.

After defining the dependency relationships, the following recipe lines specify how to generate the target — using operating system commands, each starting with a `Tab` character. Remember: `make` doesn’t care what the commands do; it just executes them. `make` compares the modification times of the target and its prerequisites. If any prerequisite is newer than the target, or if the target does not exist, then `make` executes the defined commands.

Note: `clean` is not a file — it’s just a label (like a C language label). Since there’s nothing after the colon, `make` will not automatically search for dependencies or execute it automatically. To run its associated commands, you must explicitly specify it, like `make clean`. This technique is very useful — you can define different compilation or non-compilation actions in a single makefile, such as packaging or backing up programs.

## How make works

By default, when we simply run the `make` command:
1. make searches the current directory for a file named **Makefile** or **makefile**.
2. If found, it looks for the first target in the file — in our example, that’s "edit", and treats it as the final target.
3. If the "edit" file doesn’t exist, or if any of its dependent `.o` files are newer, make executes the commands defined to build "edit".
4. If any `.o` files also don’t exist, make looks for their rules in the makefile, and builds them first. (This works recursively, like a stack process.)
5. Since your C source and header files exist, make builds the `.o` files, and then links them into the final executable "edit".

This is the entire dependency process of `make` — it recursively searches through dependency chains until it produces the first target. If an error occurs during the search, for example if a required file is missing, `make` immediately exits and reports an error. However, for errors in the defined commands (such as compilation errors), `make` doesn’t care — it only manages dependencies. If, after resolving dependencies, the required files are still missing, `make` will simply refuse to continue.

As we saw above, targets like `clean` are not connected to the first target directly or indirectly, so their commands will not run automatically. But we can explicitly run them by typing `make clean`, which removes all target files for a clean rebuild.

Therefore, in practice, if the project has already been compiled, and you modify one source file, e.g. `file.c`. Then according to the dependency rules, `file.o` will be recompiled, making it newer than `edit`, causing `edit` to be relinked (as defined by the `edit` target rule).

Similarly, if you modify `command.h`, then `kbd.o`, `command.o`, and `files.o` will all be recompiled, and `edit` will be relinked as well.


## Using Variables in a makefile

In the above example, let’s first look at the rule for `edit`:

```makefile
edit : main.o kbd.o command.o display.o \
        insert.o search.o files.o utils.o
    cc -o edit main.o kbd.o command.o display.o \
        insert.o search.o files.o utils.o
```

We can see that the string of `.o` files is repeated twice. If our project needs to add a new `.o` file, then we need to add it in two places (actually three places — another one is in `clean`). Of course, our makefile is not complicated, so adding it in two places is not tiring. But if the makefile becomes complex, we may forget one of the places that needs to be updated, causing compilation failure. Therefore, for ease of maintenance, we can use variables in the makefile. A makefile variable is just a string — it may be easier to think of it like a macro in C.

For example, we declare a variable named `objects`, `OBJECTS`, `objs`, `OBJS`, `obj`, or `OBJ`. Whatever — as long as it represents the object files. We define it at the beginning of the makefile:

```makefile
objects = main.o kbd.o command.o display.o \
     insert.o search.o files.o utils.o
```

Then, we can very conveniently use this variable in our makefile using `$(objects)`, and our improved version becomes:

```makefile
objects = main.o kbd.o command.o display.o \
    insert.o search.o files.o utils.o

edit : $(objects)
    cc -o edit $(objects)
main.o : main.c defs.h
    cc -c main.c
kbd.o : kbd.c defs.h command.h
    cc -c kbd.c
command.o : command.c defs.h command.h
    cc -c command.c
display.o : display.c defs.h buffer.h
    cc -c display.c
insert.o : insert.c defs.h buffer.h
    cc -c insert.c
search.o : search.c defs.h buffer.h
    cc -c search.c
files.o : files.c defs.h buffer.h command.h
    cc -c files.c
utils.o : utils.c defs.h
    cc -c utils.c
clean :
    rm edit $(objects)
```

So if a new `.o` file is added, we only need to modify the `objects` variable.

Regarding more topics about variables, I will cover them one by one in later **Makefile tutorials** in this series.


## Let make Deduce Automatically

GNU make is powerful. It can automatically deduce commands based on file names and dependencies, so we don’t need to write similar commands after each `.o` file, because `make` will automatically identify and deduce commands by itself.

As long as make sees a `.o` file, it will automatically add the corresponding `.c` file as a dependency. If make finds `whatever.o`, then `whatever.c` is the dependency of `whatever.o`. And `cc -c whatever.c` will also be deduced. This means our makefile no longer needs to be so long. Our new makefile looks like this:

``` makefile
objects = main.o kbd.o command.o display.o \
    insert.o search.o files.o utils.o

edit : $(objects)
    cc -o edit $(objects)

main.o : defs.h
kbd.o : defs.h command.h
command.o : defs.h command.h
display.o : defs.h buffer.h
insert.o : defs.h buffer.h
search.o : defs.h buffer.h
files.o : defs.h buffer.h command.h
utils.o : defs.h

.PHONY : clean
clean :
    rm edit $(objects)
```

This approach uses make’s “implicit rules”. In the above, `.PHONY` means `clean` is a phony target.

Regarding implicit rules and PHONY targets, I will explain them in detail in later **Makefile tutorials** in this series.



## Another Style of Makefile

Since make can automatically deduce commands, seeing all those repetitive `.o` and `.h` dependencies feels annoying. Can we gather them together? Sure. This is easy for make — it supports automatic command and file deduction. Let’s see the latest style:

``` makefile
objects = main.o kbd.o command.o display.o \
    insert.o search.o files.o utils.o

edit : $(objects)
    cc -o edit $(objects)

$(objects) : defs.h
kbd.o command.o files.o : command.h
display.o insert.o search.o files.o : buffer.h

.PHONY : clean
clean :
    rm edit $(objects)
```

Here, `defs.h` is a dependency of all target files. `command.h` and `buffer.h` are dependencies of their respective target files.

This style makes the makefile very short, but the dependency relationships become messy. It depends on your preference. Personally I don’t like this style — the dependencies become unclear, and if the project grows and new `.o` files are added, it becomes difficult to manage.

## The Rule for Cleaning the Directory

Every Makefile should have a rule to delete target files (`.o`) and executables. This is convenient for recompilation and keeps your directory clean. The usual style is:

``` makefile
clean:
    rm edit $(objects)
```

A more robust method is:

``` makefile
.PHONY : clean
clean :
    -rm edit $(objects)
```

As mentioned earlier, `.PHONY` means `clean` is a “phony target”. The hyphen before `rm` means: even if some files cannot be removed, don’t stop — continue executing the remaining commands.

And of course, the `clean` rule should not be placed at the top of the file. Otherwise, it becomes the default make target — and nobody wants that. The informal rule is: “clean is always placed at the end of the file.”

This first part of the **Makefile tutorials** series gave you a high‑level overview of Makefile basics and how `make` thinks about dependencies.

Above is an overview of a makefile and its basics. There are more details ahead — ready to go deeper? Let’s start.

## What’s Inside a Makefile?

A Makefile mainly contains five things: explicit rules, implicit rules, variable definitions, directives, and comments.

1. Explicit rules. Explicit rules specify how to generate one or more target files. They explicitly state the targets, dependencies, and commands.

2. Implicit rules. Since make supports automatic deduction, implicit rules reduce the need for writing long Makefiles. These rules are provided by make itself.

3. Variable definitions. We define variables in Makefiles, generally strings — similar to C macros. When the Makefile is executed, variables expand in place.

4. Directives. These include three types: including one Makefile in another (like #include in C), conditionally selecting parts of the Makefile (like #if), defining multi-line commands.

5. Comments. Makefiles only have line comments, like UNIX shell scripts. A comment starts with `#`, similar to `//` in C/C++. If you need to use `#` in your Makefile, escape it with a backslash like: `\#`.

Finally, remember that all commands in a Makefile must start with a `Tab` character.

## The Makefile Filename

By default, make will look for `GNUmakefile`, `makefile`, and `Makefile` in this order. Among them, it’s best to use `Makefile` because alphabetically it stays close to other important files like `README`. Avoid using `GNUmakefile` because it is only recognized by GNU make; other versions might not support it. Most versions support `makefile` and `Makefile`.

Of course, you can use other filenames such as "Make.Solaris" or "Make.Linux". To specify a Makefile manually, use `-f` or `--file`, e.g.: `make -f Make.Solaris` or `make --file Make.Linux`

You can specify multiple Makefiles by passing multiple `-f` or `--file` options.

## Including Other Makefiles
The include directive lets you `include` other Makefiles, similar to `#include` in C. The included file appears exactly where the include statement is written. Syntax:

``` Makefile
include <filenames>...
```

`<filenames>` can use shell file patterns, paths, and wildcards.

Whitespace may appear before `include`, but it must not begin with a `Tab`. `include` and `<filenames>` may be separated by one or more spaces. For example, you have `a.mk`, `b.mk`, `c.mk`, a file named `foo.make`, and a variable `$(bar)` containing `bish` and `bash`. Then the line:
``` Makefile
include foo.make *.mk $(bar)
```

is equivalent to:
```Makefile
include foo.make a.mk b.mk c.mk bish bash
```

When make starts, it will look for the included Makefiles and insert their contents into the current file. If the files don’t specify an absolute or relative path, make searches in the current directory first. If not found, make then searches the following directories:
1. Directories specified by `-I` or `--include-dir`.
2. `<prefix>/include` (usually `/usr/local/bin`), `/usr/gnu/include`, `/usr/local/include`, `/usr/include`.

Directories listed in the environment variable `.INCLUDE_DIRS`. Avoid using `-I` to reference default include paths; doing so makes `make` “forget” the normal include directories.

If an included file is not found, make gives a warning but does not immediately fail. It finishes reading the Makefile, then retries missing files. If still missing, it produces a fatal error. If you want make to ignore missing files entirely, prefix the include with a hyphen:
```Makefile
-include <filenames>...
```

This means: no matter what errors occur, don’t report them; continue.For compatibility with other versions of make, you can use `sinclude` instead of `-include`.


## Environment Variable MAKEFILES

If the environment variable `MAKEFILES` is defined in your current environment, then make will treat the value of this variable in a manner similar to `include`. The value of this variable consists of other Makefiles, separated by spaces. However, it differs from include in that the “default goal” of the Makefiles brought in through this environment variable will not take effect, and if the files defined in this environment variable contain errors, make will also ignore them.

But here I still recommend not using this environment variable, because once this variable is defined, every time you use make, all Makefiles will be affected by it — and this is definitely not something you want to see. I mention this only to tell you that if your Makefile behaves weirdly, you can check whether this variable is defined in your current environment.

## How make Works

When GNU make runs, its execution steps are as follows (other versions of make behave similarly):
1. Read in all Makefiles.
2. Read in the other Makefiles specified by include.
3. Initialize the variables defined in the files.
4. Deduce implicit rules and analyze all rules.
5. Create dependency chains for all target files.
6. Based on the dependency relationships, decide which targets need to be regenerated.
7. Execute the commands to build them.

Steps 1–5 make up the first stage, and 6–7 make up the second stage.
During the first stage, if a defined variable is used, make will expand the variable at the place where it is used. However, make does not expand everything immediately — it uses a delayed expansion strategy. If a variable appears in a dependency rule, then the variable will only be expanded when this dependency rule is determined to be used.



