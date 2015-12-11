---
title: Fast and correct incremental building with pluto
layout: default
---

Build systems are used in all but the smallest software projects to invoke the
right build tools on the right files in the right order. A build system must
satisfy two central properties. First, a build system must guarantee correct
builds, which means that generated artifacts consistently reflect the latest
source artifacts. Second, a build system must be efficient, which means that it
incrementally rechecks and rebuilds as few build steps as possible.

### Example

Consider the following build scenario for a Java preprocessor that
uses ANTLR for parsing the non-preprocessed source code.

1. Checkout preprocessor source code from remote repository.
2. Read configuration file declaring which Java version the preprocessor
   targets.
3. Build ANTLR parser for that Java version.
	1. Download ANTLR binary.
    2. Download ANTLR grammar for correct Java version.
    3. Run ANTLR binary to compile the ANTLR grammar into Java code.
    4. Compile Java code.
4. Compile preprocessor source code against parser.
5. Package preprocessor together with parser into JAR file.

There are many source artifacts involved in this build, such as the code in the
remote repository, the code in the local working copy, the configuration file,
the ANTLR binary, or the ANTLR grammar. Correctness requires that the build
system rebuilds the generated artifacts whenever any one of the source artifacts
changes. Efficiency requires that the build system reexecutes as few build steps
as possible. For example, if the code in the remote repository changes but the
Java version in the configuration file remains unchanged, the build system
should only execute steps 1, 2, 4, and 5.

### Dynamic dependencies needed

Almost all existing build systems fail on this example and either provide
inefficient or incorrect building. The reason is that it is only possible to
determine whether to execute step 3 after executing step 1 and 2, that is, after
fetching the configuration file and extracting the version number. Thus, the
build script has a *dynamic dependency* on step 3. This is in contrast to static
dependencies that are hard-coded in the build script. Existing build systems
only support static dependencies, which they use to construct a build schedule
before executing any build steps. Since the build schedule cannot change during
build execution, existing build systems either inefficiently execute step 3 even
when the Java version does not change or incorrectly only execute step 3 on the
first run, reusing the parser even when the Java version changes.

Not only are dynamic dependencies necessary for the example scenario above, it
is also easier to derive precise dynamic dependencies. For example, when
compiling the preprocessor source code in step 4 above, the code repository may
contain source files that are not used by the processor. With static
dependencies, we have to make the required source files explicit in the build
script or apply a static dependency analyzer to generate this information before
the build execution starts. With dynamic dependencies, we can simply start the
Java compiler and inspect its verbose output to determine the set of required
source files. In practice, dynamic dependencies make precise dependency analysis
a lot easier and yield less overapproximation (superfluous dependencies) and
less underapproximation (missing dependencies).

### The pluto build system

We have developed a new build system called *pluto* that supports dynamic build
dependencies. In pluto, each build step consists of a build function that can
declare dependencies on files and other build steps during its execution through
a simple API. When a build step finishes, pluto stores the dependency list of
the build step to detect changes of required artifacts in subsequent runs.
However, in contrast to other build systems, pluto combines change detection and
the execution of build steps into a single traversal of the dependency
graph. This traversal only initiates the reexecution of those build steps that
(i) are still required and (ii) depend on a file that was changed since the
previous build. We have formally verified the correctness and optimality of
pluto's incremental rebuilding algorithm.

We have developed pluto as a Java API where users can define build steps by
implementing a simple Java interface. Using a full-fledged programming language
like Java for build scripts has various positive design implications. First and
maybe most importantly, this design makes it easy to debug a build script using
a stock IDE such as Eclipse. Second, build-script authors can use all features
of Java and are not bound to a low-level language such as Bash. Third, our API
makes it easy to parameterize build steps over run-time Java
values. Parameterized build steps can often be reused across build
scripts. Fourth, the type system of Java provides a statically typed interface
to build steps and ensures that build steps are invoked on well-typed arguments
only.


### Team
