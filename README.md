# auditmake

Check makefiles for missing prerequisites (dependencies).

## INTRODUCTION

In any sizable project, the makefiles are almost always wrong in subtle ways.
Problems can remain undiagnosed indefinitely.  People tend to sense the
problems exist and develop a mistrust of the build system resorting to
frequently doing "make clean" wasting time.  ("My program was behaving oddly,
so I did a `make clean;make` and now it's working right.")

This tool uses GNU make and strace to find missing prerequisites in makefiles.

As a simple example:

	foo:
	     cp bar foo

The rule uses bar when building foo but doesn't list bar as a prerequisite.
The correct makefile would be:

	foo: bar
	     cp bar foo

## REQUIREMENTS

  * GNU make 3.82.  auditmake uses the `.ONESHELL` feature introduced in GNU
    make version 3.82.

    Although 3.82 was released several years ago (July 2010), it introduced a
    few backward incompatibilities and so you might not have it installed.
    http://lists.gnu.org/archive/html/info-gnu/2010-07/msg00023.html

    Run the command `make -v` to determine what version of make is installed.

    To install 3.82:

		wget http://ftp.gnu.org/gnu/make/make-3.82.tar.bz2
		tar xjf make-3.82.tar.bz2
		cd make-3.82
		./configure --prefix=/usr/local/make-3.82 # or wherever you prefer
		make
		sudo make install

  * strace

  * perl

## USAGE

Put the following at the top of your makefile:

	SRCROOT=...
	ifdef AUDITMAKE
	  ifeq (,$(filter shortest-stem,$(.FEATURES)))
	    $(error You need GNU make version 3.82 or later, you have $(MAKE_VERSION))
	  endif
	  .ONESHELL:
	  SHELL=auditmake
	  .SHELLFLAGS=$(SRCROOT) $@ $^
	endif

Set the `SRCROOT` line to be the root of your source code tree.  auditmake
ignores files that are outside this tree.

The auditmake script should be in your `PATH`.  Or else change the `SHELL=`
line to be an absolute path name.

Many build systems generate `*.d` files listing the header file dependencies
of C/C++ source files.  Normally, these `*.d` files are removed by a `make
clean`, but this script needs them.  So, you should do a full build, save the
`*.d` files, `make clean`, restore the `*.d` files, then run this script.  For
example:

	  make
	  find -name \*.d | tar -czf saved.tar.gz --files-from -
	  make clean
	  tar -xzf saved.tar.gz
	  export MAKE=/usr/local/bin/make-3.82/bin/make
	  $MAKE AUDITMAKE=1

If auditmake finds any missing prerequisites, you'll see output like the
following:

	auditmake: foo.c used the following without specifying as prerequisite:
	  bar.h
	  baz.h

## HOW IT WORKS

Via the `SHELL` and `.SHELLFLAGS` variables, `make` passes to auditmake
information about what target is to be built, what direct prerequisites were
listed, and what commands should be executed to build the target.  auditmake
executes the commands via strace and then analyzes strace's output.

auditmake watches for the following syscalls:

  * open().  Excludes O_DIRECTORY, O_TRUNC and O_WRONLY because these are not
    inputs.

  * openat(AT_FDCWD,...).  Same as open(2).

  * getcwd(), clone(), chdir().  auditmake uses this information to convert
    relative path names to absolute.

## LIMITATIONS

There are probably more syscalls that auditmake should watch (e.g., mmap()).

strace version 4.5.20 on 64-bit hosts can't trace 32-bit executables.  You see
the following message:

	[ Process PID=7569 runs in 32 bit mode. ]

strace doesn't fail, it just doesn't generate the trace.
