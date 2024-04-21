---
title: 'Nix pill 12: the inputs design pattern'
date: 2014-08-28T08:08:00.001-07:00
draft: false
url: /2014/08/nix-pill-12-inputs-design-pattern.html
tags: 
- nixpkgs
- nix
- nixpills
- nixos
---

Welcome to the 12th Nix pill. In the previous [11th pill](http://lethalman.blogspot.it/2014/08/nix-pill-11-garbage-collector.html) we stopped packaging and cleaned up the system with the garbage collector.

  
We restart our packaging, but we will improve a different aspect. We only packaged an hello world program so far, what if we want to create a repository of multiple packages?  
  

### Repositories in Nix

  
Nix is a tool for build and deployment, it does not enforce any particular repository format. A repository of packages is the main usage for Nix, but not the only possibility. See it more like a consequence due to the need of organizing packages.  
Nix is a language, and it is powerful enough to let you choose the format of your own repository. In this sense, it is not declarative, but functional.  
There is no preset directory structure or preset packaging policy. It's all about you and Nix.  
  
The [nixpkgs](http://github.com/NixOS/nixpkgs) repository has a certain structure, which evolved and evolves with the time. Like other languages, Nix has its own history and therefore I'd like to say that it also has its own design patterns. Especially when packaging, you often do the same task again and again except for different software. It's inevitable to identify patterns during this process. Some of these patterns get reused if the community thinks it's a good way to package the software.  
  
Some of the patterns I'm going to show do not apply only to Nix, but to other systems of course.  
  

### The single repository pattern

  
Before introducing the "inputs" pattern, we can start talking about another pattern first which I'd like to call "single repository" pattern.  
Systems like Debian scatter packages in several small repositories. Personally, this makes it hard to track interdependent changes and to contribute to new packages.  
Systems like Gentoo instead, put package descriptions all in a single repository.  
  
The nix reference for packages is [nixpkgs](https://github.com/NixOS/nixpkgs), a single repository of all descriptions of all packages. I find this approach very natural and attractive for new contributions.  
  
From now on, we will adopt this technique. The natural implementation in Nix is to create a top-level Nix expression, and one expression for each package. The top-level expression imports and combines all expressions in a giant attribute set with name -> package pairs.  
  
But isn't that heavy? It isn't, because Nix is a [lazy](http://en.wikipedia.org/wiki/Lazy_evaluation) language, it evaluates only what's needed! And that's why nixpkgs is able to maintain such a big software repository in a giant attribute set.  
  

### Packaging graphviz

  
We have packaged GNU hello world, I guess you would like to package something else for creating at least a repository of two projects :-) . I chose graphviz, which uses the standard autotools build system, requires no patching and dependencies are optional.  
  
Download graphviz from [here](http://www.graphviz.org/pub/graphviz/stable/SOURCES/graphviz-2.38.0.tar.gz). The graphviz.nix expression is straightforward:  
```
let
  pkgs = import <nixpkgs> {};
  mkDerivation = import ./autotools.nix pkgs;
in mkDerivation {
  name = "graphviz";
  src = ./graphviz-2.38.0.tar.gz;
}

```

Build with nix-build graphviz.nix and you will get runnable binaries under result/bin. Notice how we did reuse the same autotools.nix of hello.nix. Let's create a simple png:  
```
$ echo 'graph test { a -- b }'|result/bin/dot -Tpng -o test.png
Format: "png" not recognized. Use one of: canon cmap \[...\]

```
Oh of course... graphviz can't know about png. It built only the output formats it supports natively, without using any extra library.  
  
I remind you, in autotools.nix there's a buildInputs variablewhich gets concatenated to baseInputs.  That would be the perfect place to add a build dependency. We created that variable exactly for this reason to be overridable from package expressions.  
  
This 2.38 version of graphviz has several plugins to output png. For simplicity, we will use libgd.  
  

### Digression about gcc and ld wrappers

  
The gd, jpeg, fontconfig and bzip2 libraries (dependencies of gd) don't use pkg-config to specify which flags to pass to the compiler. Since there's no global location for libraries, we need to tell gcc and ld where to find includes and libs.  
  
The nixpkgs provides gcc and binutils, and we are using them for our packaging. Not only, it also [provides wrappers](http://nixos.org/nixpkgs/manual/#ssec-setup-hooks) for them which allow passing extra arguments to gcc and ld, bypassing the project build systems:  

*   NIX\_CFLAGS\_COMPILE: extra flags to gcc at compile time
*   NIX\_LDFLAGS: extra flags to ld

What can we do about it? We can employ the same trick we did for PATH: automatically filling the variables from buildInputs. This is the relevant snippet of setup.sh:  
```
for p in $baseInputs $buildInputs; do
  if \[ -d $p/bin \]; then
    export PATH="$p/bin${PATH:+:}$PATH"
  fi
  if \[ -d $p/include \]; then
    export NIX\_CFLAGS\_COMPILE="-I $p/include${NIX\_CFLAGS\_COMPILE:+ }$NIX\_CFLAGS\_COMPILE"
  fi
  if \[ -d $p/lib \]; then
    export NIX\_LDFLAGS="-rpath $p/lib -L $p/lib${NIX\_LDFLAGS:+ }$NIX\_LDFLAGS"
  fi
done

```
Now by adding derivations to buildInputs, will add the lib, include and bin paths automatically in setup.sh.  
  
The -rpath flag in ld is needed because at runtime, the executable must use exactly that version of the library.  
If unneeded paths are specified, the fixup phase will shrink the rpath for us!  
  

### Completing graphviz with gd

  
Finish the expression for graphviz with gd support (note the use of the **with** expression in **buildInputs** to avoid repeating pkgs):  
```
let
  pkgs = import <nixpkgs> {};
  mkDerivation = import ./autotools.nix pkgs;
in mkDerivation {
  name = "graphviz";
  src = ./graphviz-2.38.0.tar.gz;
  buildInputs = with pkgs; \[ gd fontconfig libjpeg bzip2 \];
}

```

Now you can create the png! Ignore any error from fontconfig, especially if you are in a chroot.  
  

### The repository expression

  
Now that we have two packages, what's a good way to put them together in a single repository? We do something like nixpkgs does. With nixpkgs, we import it and then we peek derivations by accessing the giant attribute set.  
For us nixers, this a good technique, because it abstracts from the file names. We don't refer to a package by REPO/some/sub/dir/package.nix but by importedRepo.package (or pkgs.package in our examples).  
  
Create a default.nix in the current directory:  
```
{
  hello = import ./hello.nix; 
  graphviz = import ./graphviz.nix;
}

```
Ready to use! Try it with nix-repl:  
```
$ nix-repl
nix-repl> :l default.nix
Added 2 variables.
nix-repl> hello
«derivation /nix/store/dkib02g54fpdqgpskswgp6m7bd7mgx89-hello.drv»
nix-repl> graphviz
«derivation /nix/store/zqv520v9mk13is0w980c91z7q1vkhhil-graphviz.drv»

```
With nix-build:  
```
$ nix-build default.nix -A hello
\[...\]
$ result/bin/hello
Hello, world!

```
The -A argument is used to access an attribute of the set from the given .nix expression.  
**Important**: why did we choose the default.nix? Because when a directory (by default the current directory) has a default.nix, that default.nix will be used (see import [here](http://nixos.org/nix/manual/#ssec-builtins)). In fact you can run nix-build -A hello without specifying default.nix.  
For pythoners, it is similar to \_\_init\_\_.py.  
  
With nix-env, to install the package in your user environment:  
```
$ nix-env -f . -iA graphviz
\[...\]
$ dot -V

```
The -f option is used to specify the expression to use, in this case the current directory, therefore ./default.nix.  

The -i stands for installation.

The -A is the same as above for nix-build.  
  
We reproduced the very basic behavior of nixpkgs.  
  

### The inputs pattern

  
After a long preparation, we finally arrived. I know you have a big doubt in this moment. It's about the hello.nix and graphviz.nix. They are very, very dependent on nixpkgs:  

*   First big problem: they import nixpkgs directly. In autotools.nix instead we pass nixpkgs as an argument. That's a much better approach.
*   Second problem: what if we want a variant of graphviz without libgd support?
*   Third problem: what if we want to test graphviz with a particular libgd version?

The current answer to the above questions is: change the expression to match your needs (or change the callee to match your needs).

With the inputs pattern, we choose to give another answer: let the user change the inputs of the expression (or change the caller to pass different inputs).

  

By inputs of an expression, we refer to the set of derivations needed to build that expression. In this case:

*   mkDerivation from autotools. Recall that mkDerivation has an implicit dependency on the toolchain.
*   libgd and its dependencies.

The src is also an input but it's pointless to change the source from the caller. For version bumps, in nixpkgs we prefer to write another expression (e.g. because patches are needed or different inputs are needed).

  

**Goal**: make package expressions independent of the repository.  
  

How do we achieve that? The answer is simple: **use functions to declare inputs for a derivation**. Doing it for graphviz.nix, will make the derivation independent of the repository and customizable:

```
{ mkDerivation, gdSupport ? true, gd, fontconfig, libjpeg, bzip2 }:

mkDerivation {
  name = "graphviz";
  src = ./graphviz-2.38.0.tar.gz;
  buildInputs = if gdSupport then \[ gd fontconfig libjpeg bzip2 \] else \[\];
}

```

I recall that "{...}: ..." is the syntax for defining [functions](http://nixos.org/nix/manual/#idm47361539166560) accepting an attribute set as argument.  
  
We made gd and its dependencies optional. If gdSupport is true (by default), we will fill buildInputs and thus graphviz will be built with gd support, otherwise it won't.  
Now back to default.nix:  
```
let
  pkgs = import <nixpkgs> {};
  mkDerivation = import ./autotools.nix pkgs;
in with pkgs; {
  hello = import ./hello.nix { inherit mkDerivation; }; 
  graphviz = import ./graphviz.nix { inherit mkDerivation gd fontconfig libjpeg bzip2; };
  graphvizCore = import ./graphviz.nix {
    inherit mkDerivation gd fontconfig libjpeg bzip2;
    gdSupport = false;
  };
}

```

So we factorized the import of nixpkgs and mkDerivation, and also added a variant of graphviz with gd support disabled. The result is that both hello.nix (exercise for the reader) and graphviz.nix are independent of the repository and customizable by passing specific inputs.  
If you wanted to build graphviz with a specific version of gd, it would suffice to pass gd = ...;.  
If you wanted to change the toolchain, you may pass a different mkDerivation function.  
  
Clearing up the syntax:  

*   In the end we return an attribute set from default.nix. With "let" we define some local variables.
*   We bring pkgs into the scope when defining the packages set, which is very convenient instead of typing everytime "pkgs".
*   We import hello.nix and graphviz.nix, which will return a function, and call it with a set of inputs to get back the derivation.
*   The "inherit x" syntax is equivalent to "x = x". So "inherit gd" here, combined to the above "with pkgs;" is equivalent to "x = pkgs.gd".

You can find the whole repository at the [pill 12 gist](https://gist.github.com/lethalman/734b168a0258b8a38ca2).

  

### Conclusion

  

The "inputs" pattern allows our expressions to be easily customizable through a set of arguments. These arguments could be flags, derivations, or whatelse. Our package expressions are functions, don't think there's any magic in there.  
  
It also makes the expressions independent of the repository. Given that all the needed information is passed through arguments, it is possible to use that expression in any other context.  
  

### Next pill

  
...we will talk about the "callPackage" design pattern. It is tedious to specify the names of the inputs twice, once in the top-level default.nix, and once in the package expression. With callPackage, we will implicitly pass the necessary inputs from the top-level expression.  
  
Pill 13 is available for [reading here](http://lethalman.blogspot.it/2014/09/nix-pill-13-callpackage-design-pattern.html).  
  

To be notified about the new pill, stay tuned on [#NixPills](https://twitter.com/search?src=typd&q=%23NixPills), follow [@lethalman](https://twitter.com/lethalman) or subscribe to the [nixpills rss](http://lethalman.blogspot.com/feeds/posts/default/-/nixpills).