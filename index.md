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

### Illustrating example

Consider the following build scenario for a Java preprocessor that
uses ANTLR for parsing the non-preprocessed source code.

1. Checkout preprocessor source code from remote repository.
2. Extract Java version targeted by the preprocessor from a configuration file.
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
system rebuilds the generated artifacts whenever any one of these source
artifacts changes. Efficiency requires that the build system reexecutes as few
build steps as possible. For example, if the code in the remote repository
changes but the Java version in the configuration file remains unchanged, the
build system should only execute steps 1, 2, 4, and 5. Note that the
configuration file may require non-trivial parsing to determine the targeted
Java version.

### Dynamic dependencies needed!

Almost all existing build systems fail on this example and either provide
inefficient or incorrect building. The reason is that it is only possible to
determine whether to execute step 3 after executing step 1 and 2, that is, after
fetching the configuration file and extracting the version number. Thus, the
build script has a *dynamic dependency* on step 3. This is in contrast to static
dependencies that are hard-coded in the build script. Existing build systems
only support static dependencies, which they use to construct schedule the
execution of build steps. Since the build schedule cannot change during build
execution, existing build systems either inefficiently execute step 3 even when
the Java version does not change or incorrectly only execute step 3 on the first
run, reusing the parser even when the Java version changes. Dynamic dependencies
allow us to decide after step 2 finished for which version of Java step 3 should
build a parser.

Dynamic dependencies are not only necessary for the example scenario above, it
is also easier to derive precise dynamic dependencies than it is to approximate
static dependencies. For example, when compiling the preprocessor source code in
step 4 above, the code repository may contain source files that are not used by
the preprocessor implementation. With static dependencies, we have to make the
required source files explicit in the build script or apply a static dependency
analyzer to generate this information before the build execution starts. With
dynamic dependencies, we can simply start the Java compiler and inspect its
verbose output to determine the set of required source files. In practice,
dynamic dependencies make precise dependency analysis a lot easier and yield
less overapproximation (less superfluous dependencies) and less
underapproximation (less missing dependencies).

### The pluto build system

We have developed a new build system called
[*pluto*](http://pluto-build.github.io/) that supports dynamic build
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
implementing a simple Java interface. For illustration, the script of build step
3 of our example is
[available online](https://github.com/pluto-build/build-examples/blob/master/src/build/pluto/buildexamples/BuildANTLRParser.java)
for inspection.
Using a full-fledged programming language like Java for build scripts has
various positive design implications. First and maybe most importantly, Java
makes it possible to debug a build script using any Java IDE. Second,
build-script authors can use all features of Java and are not bound to a
low-level language such as Bash. Third, our API makes it easy to parameterize
build steps over run-time Java values. Parameterized build steps can often be
reused across build scripts. Fourth, the type system of Java provides a
statically typed interface for using build steps and ensures that build steps
are invoked on well-typed arguments only.

In summary, pluto provides the following features:

* Dynamic dependencies on files and other build steps.
* A Java API for debuggable and reusable build steps.
* Dependency injection into build steps to avoid hard-coded dependencies.
* Semantic file stamps for avoiding rebuilds after irrelevant file changes.
* Automatic metadependencies that ensure a rebuild after the build script changes.
* Detection of build cycles and customizable cycle handling.


### pluto is fast

We have used pluto to develop build scripts for a few projects so far. In
particular, we have migrated the existing Ant build script of the
[Spoofax IDE](http://spoofax.org) to pluto. Using pluto, we achieved speedups of
4x-50x for rebuilding after a typical program change. The speedup depends on how
many downstream build steps a file change invalidates. The fine-grained
dependency tracking of pluto also revealed hidden dependencies in the original
build script that led to incorrect rebuilds. Pluto automatically detects such
hidden dependencies and guarantees correct rebuilds in all cases.

We have used pluto to develop numerous reusable build steps that perform
fine-grained dependency tracking out of the box. The provided build steps range
from Java compilation and JUnit testing to dependency resolution via git and
Maven. Our experience shows that it is easy to develop new build steps and to
make them reusable.

### Resources on pluto

* Pluto website [http://pluto-build.github.io/](http://pluto-build.github.io/)
* Pluto source code
  [https://github.com/pluto-build/](https://github.com/pluto-build/)
* Academic article on pluto:
  [A Sound and Optimal Incremental Build System with Dynamic Dependencies](http://erdweg.org/publications/pluto-incremental-build.pdf)
  by Sebastian Erdweg, Moritz Lichter, and Manuel Weiel. Published at the
  Conference on Object-Oriented Programming, Systems, Languages, and
  Applications (OOPSLA).

