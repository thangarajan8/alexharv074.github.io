---
layout: post
title: "GNU make for DevOps engineers"
date: 2019-12-26
author: Alex Harvey
tags: make
---

GNU make is a tool designed for compiling executable programs and libraries from source code, such as C programs. The original make was written in 1976 at Bell Labs by Stuart Feldman. It has made a comeback in recent years, outside the world of C and other software development, as a build automation tool used by DevOps engineers to launch Docker containers and run automated tests.

In this post, I present a reference and tutorial for DevOps engineers who want to deepen their knowledge of GNU make.

- ToC
{:toc}

## Makefile

Make is about writing Makefiles. A file named `Makefile` typically will live in the top level of a software project and this file tells make what to do. It is expected to contain a list of rules for compiling software or running other build tasks.

## Rules

### Structure of a rule

The _rules_ in a Makefile have the following form:

```text
target [target ...]: [prerequisite ...]
	[recipe]
	...
	...
```

Note that indentation is by TAB characters, although this behaviour is configurable via the `.RECIPEPREFIX` variable.

### Simplest example

The simplest example of a rule, although artificial, might be a rule for creating a file "hello". Consider a Makefile with the following content:

```make
hello:
	touch hello
```

If you type make the first time:

```text
▶ make
touch hello
▶ ls -l hello
-rw-r--r--  1 alexharvey  staff  0 26 Dec 17:51 hello
```

A file "hello" is created. Then type make a second time:

```text
▶ make
make: `hello' is up to date.
```

Make says that "hello" is up to date because there is already a file named "hello".

So, the one thing that make is designed to do well is create or "make" files. Keep this in mind because, as DevOps engineers, we are typically, in a way, abusing make's other features - and these other features are actually kinda hacky when you think about it! - to be a general purpose orchestration tool!

### Targets

A _target_ is supposed to be the name of the file or files that are generated, such as compiled executables or object files. In the example above, the target is a file "hello" that is generated by the recipe "touch hello".

### Prerequisites

The final part of a rule is the list of one or more _prerequisites_. (Some manuals refer these these as _components_.) These would be files like object files that are used as inputs to the compilation process, although strictly speaking they are a list of other targets whose recipes will be run as prerequisites.

Note that the prerequisites in make are quite similar to `depends_on` in Terraform and `require` in Puppet. Yes, make is also a declarative DSL.

Now consider a more complex Makefile:

```make
hello: world
	touch hello

world:
	touch world

clean:
	rm -f hello world
```

The "hello" rule now has a prerequisite "world". If I run make:

```text
▶ make
touch world
touch hello
```

As can be seen, the "world" rule has been executed before the "hello" rule's recipe. And I can clean up now using the "clean" target:

```text
▶ make clean
rm -f hello world
```

### Default goal

By default, as we can see, make processes the first rule it finds in the Makefile (excluding any whose targets begin with "."). This first rule is called the _default goal_. If I modify this Makefile as follows:

```make
hello:
	touch hello

world:
	touch world

clean:
	rm -f hello world
```

Typing "make" on its own just runs the default goal:

```text
▶ make
touch hello
```

To run the "world" target:

```text
▶ make world
touch world
```

### Configuring the default goal

The default goal is configurable using the `.DEFAULT_GOAL` variable. If I want the "world" target instead to be the default goal:

```make
.DEFAULT_GOAL := world

hello:
	touch hello

world:
	touch world

clean:
	rm -f hello world
```

Then:

```text
▶ make
touch world
```

### Phony targets

I have been using a "clean" rule for cleaning up:

```make
clean:
	rm -f hello world
```

This rule is different from the others in that it doesn't create a file. In make's terminology, a target that performs an action that doesn't result in a file being created is a "phony target".

Now try this:

```text
▶ make hello
touch hello
▶ touch clean
▶ make clean
make: `clean' is up to date.
```

So make detected a file in the current directory "clean" and decided that we already compiled a binary file "clean" and there is nothing to do here. Make doesn't want to overwrite our files after all.

Most of the time this won't happen, because there is usually no reason for files to exist that conflict with the names of make targets. But it would certainly be confusing if it did happen, especially if someone is not deeply familiar with make.

To avoid this, a special built-in make target `.PHONY` can be used to explicitly declare a phony target:

```make
.PHONY: clean

clean:
	rm -f hello world
```

Then:

```text
▶ make clean
rm -f hello world
```

My recommendation would be to always declare all phony targets explicitly both for clarity and also to avoid the edge-case when a file of the same name coincidentally exists.

### Putting it all together

In this section I present a real world example of compiling a C program, a text editor called "edit". While the post is for DevOps engineers, I think it is helpful all the same to see make used for compiling C programs. This is based on the example from the GNU make manual. Here it is:

```make
.PHONY: clean

objects := main.o kbd.o command.o display.o \
	insert.o search.o files.o utils.o

edit: $(objects)
	cc -o edit $(objects)

main.o: main.c defs.h
	cc -c main.c

kbd.o: kbd.c defs.h command.h
	cc -c kbd.c

command.o: command.c defs.h command.h
	cc -c command.c

display.o: display.c defs.h buffer.h
	cc -c display.c

insert.o: insert.c defs.h buffer.h
	cc -c insert.c

search.o: search.c defs.h buffer.h
	cc -c search.c

files.o: files.c defs.h buffer.h command.h
	cc -c files.c

utils.o: utils.c defs.h
	cc -c utils.c

clean:
	rm edit main.o kbd.o command.o display.o \
		insert.o search.o files.o utils.o
```

The GNU make manual goes on to further "simplify" this example by using "implicit rules" and other features. These other features are unlikely to matter outside the world of C programming, so I present the Makefile in the above form.

Note that, as mentioned already, make is a declarative DSL much like Puppet and Terraform, where an end-state is declared and make figures out the ordering of things by building a dependency graph.

## Two phases of make

Also just like Puppet and Terraform, make also works in two distinct phases:

1. In the first phase, make reads in its Makefile (and included makefiles), sets all variables, and constructs a dependency graph of targets and their prerequisites.
1. Then, in the second phase, it figures out which targets need to be rebuilt and then actually rebuilds them.

Occasionaly - e.g. when using conditionals - it is necessary to be aware of the two phases.

## Recipe echoing

Another feature I want to cover at the beginning is _recipe echoing_, since that makes it easier for me to test code and write clean code examples. Normally, make prints each line of its recipes before they are executed. This is called recipe echoing. For example:

```make
test:
	echo testing 1, 2, 3
```

This leads to some ugly output:

```text
▶ make test
echo testing 1, 2, 3
testing 1, 2, 3
```

But echoing can be disabled by prefixing the line with a `@` symbol. Thus:

```make
test:
	@echo testing 1, 2, 3
```

This leads to much less ugly output, and perfect for testing:

```text
▶ make test
testing 1, 2, 3
```

## Variables

### Assigning and referencing variables

The reason that Makefiles are likely to be confusing to the uninitiated is that they generally consist of Bash code intermixed with Makefile DSL code that use similar but different notation. First up are variables. A variable in a Makefile is assigned using notation like `foo := bar` or `foo = bar` (see next section for the difference) and is referenced using notation `$(foo)` or `${foo}`. And if the variable has only one character it can also be referenced as `$f`. Confusing!

Assign a variable:

```make
foo := bar
test:
	touch $(foo)
	touch ${foo}
	touch $foo
```

If I run that:

```text
▶ make foo
touch bar
touch bar
touch oo
```

Notice the last line "touch oo". That's because `$foo` expands as `$f` - a variable that I didn't set - plus `oo`.

### Recursively expanded versus simply expanded variables

#### Recursively expanded

Even more confusing is make's concept of the _recursively expanded variable_. Notice here that variables can be referenced in the Makefile before they have been declared:

```make
foo = $(bar)
bar = $(baz)
baz = Huh?

all:
	touch $(foo)
```

That leads rather counter-intuitively to this:

```text
▶ make all
touch Huh?
```

#### Simply expanded

Put simply, if you want your variables to behave the same way they do in other programming languages, just always use _simply expanded variables_. These are assigned using `foo := bar` notation. Here is an example showing these variables behaving like in any other language:

```make
x := foo
y := $(x) bar
x := later
all:
	touch $(x) $(y)
```

That leads to:

```text
▶ make all
touch later foo bar
```

So I am going to always use simply expanded variables and I doubt that I am going to find a reason to ever not do this!

### Setting a default

Like some other languages, make also allows a variable to be assigned a value only if it does not already have a value. That notation is `foo ?= bar`. An example:

```make
foo := bar
foo ?= baz
all:
	touch $(foo)
```

That leads to:

```text
▶ make all
touch bar
```

### Appending to a variable

Make also supports an append operator, using notation `foo += bar`:

```make
foo := foo
foo += bar
all:
	touch $(foo)
```

Leads to:

```text
▶ make all
touch foo bar
```

Note the space is added too.

### Environment variables

Environment variables appear within make with the same names and values. Here is an example:

```make
all:
	touch $(FOO)
```

Then:

```text
▶ FOO=bar make all
touch bar
```

## Calling a Bash one-liner

Make has many built-in functions, most of them for transforming text. Most of the time, however, it would be obfuscating to use make's functions when the same outcome can be achieved using a Bash one-liner.

To insert the result of a Bash one-liner in a make variable, use the `shell` function. Here is an example:

```make
uname := $(shell uname -s)

test:
	@echo "You are on platform $(uname)"
```

To run that:

```text
▶ make
You are on platform Darwin
```

## Conditionals in make

### Using directives

Text can be declared conditionally in Makefiles using the conditional syntax. That syntax is basically this:

```text
[conditional-directive]
  text
else [conditional-directive]
  text
else
  text
endif
```

Then there are four conditional directives that test different conditions:

|directive|description|
|---------|-----------|
|`ifeq (arg1, arg2)`|Expand all variable references in arg1 and arg2 and compare them. If they are identical, the conditional evaluates to true; otherwise false. Note that the surrounding brackets can be omitted.|
|`ifneq (arg1, arg2)`|Similar to `ifeq` but means "if not equal".|
|`ifdef variable`|Returns true if the named variable has a non-empty value.|
|`ifndef variable`|Returns true if the named variable is has an empty value.|

Here is a simple example:

```make
test:
ifeq ($(SELECT),a)
	@echo "Choosing path a"
else
	@echo "Choosing path b"
endif
```

And to test that:

```text
▶ SELECT=a make test
Choosing path a
```

Note that conditionals are operating in make's first phase. That is, they are code-generating the targets to be built in phase two. Thus, the above could be rewritten like this and have the same effect:

```make
ifeq ($(SELECT),a)
test:
	@echo "Choosing path a"
else
test:
	@echo "Choosing path b"
endif
```

### Using functions

There are also three conditional functions that can be used:

|function|description|
|--------|-----------|
|`$(if condition,then-part[,else-part])`|If the condition evaluates to a non-empty string, do the then-part; else do the else-part.|
|`$(or condition1,condition2[,condition3 ...])`|Select the first condition from condition1, condition2 etc that evaluates to a non-empty string.|
|`$(and condition1,condition2[,condition3 ...])`|If an argument expands to an empty string the processing stops and the result of the expansion is the empty string. Otherwise, the result of the expansion is the expansion of the last argument.|

Here is an example:

```make
uname := $(shell uname -s)
is_darwin := $(filter Darwin,$(uname))
sed := $(if $(is_darwin),/usr/local/bin/gsed,sed)

test:
	@echo "Your sed is $(sed)"
```

Testing that:

```text
▶ make
Your sed is /usr/local/bin/gsed
```

## Command line arguments

Make understands a few command line arguments that a make user probably needs to be aware of.

|Option|Explanation|
|======|===========|
|`-f`, `--file`|If you wish to specify a Makefile other than the one in the default path, use the `-f` option.|
|`-n`, `--just-print`, `--dry-run`, `--recon`|“No-op”. Causes make to print the recipes that are needed to make the targets up to date, but not actually execute them.|

## Features not covered

As I mentioned earlier, make is a tool for compiling and building programs written in old languages like C, C++, Fortran, Pascal etc. In this brief tutotial, I have covered the features that I think are useful or potentially useful to a DevOps engineer, who will be using make as an orchestration tool and test runner. So I have omitted a raft of features including most of its functions, most of its special variables, its implicit rules - built-in magic rules use for compilation.

A complete index of all make's functions, variables and directives is [here](https://www.gnu.org/software/make/manual/html_node/Name-Index.html#Name-Index).

I hope this has been useful. If you find there is a feature I haven't covered that you think I should have, do let me know and I will update!
