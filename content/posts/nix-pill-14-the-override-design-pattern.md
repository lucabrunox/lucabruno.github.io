---
title: 'Nix pill 14: the override design pattern'
date: 2014-09-10T04:38:00.000-07:00
draft: false
url: /2014/09/nix-pill-14-override-design-pattern.html
tags: 
- nixpkgs
- nix
- nixpills
- nixos
---

Welcome to the 14th Nix pill. In the previous [13th pill](http://lethalman.blogspot.it/2014/09/nix-pill-13-callpackage-design-pattern.html) we have introduced the callPackage pattern, used to simplify the composition of software in a repository.

  
The next design pattern is less necessary but useful in many cases and it's a good exercise to learn more about Nix.  
  

### About composability

  
Functional languages are known for being able to compose functions. In particular, you gain a lot from functions that are able to manipulate the original value into a new value having the same structure. So that in the end we're able to call multiple functions to have the desired modifications.  
  
In Nix we mostly talk about **functions** that accept inputs in order to return **derivations**. In our world we want nice utility functions that are able to manipulate those structures. These utilities add some useful properties to the original value, and we must be able to apply more utilities on top of it.  
  
For example let's say we have an initial derivation drv and we want it to be a drv with debugging information and also to apply some custom patches:  
```
debugVersion (applyPatches \[ ./patch1.patch ./patch2.patch \] drv)

```
The final result will be still the original derivation plus some changes. That's both interesting and very different from other packaging approaches, which is a consequence of using a functional language to describe packages.  
  
Designing such utilities is not trivial in a functional language that is not statically typed, because understanding what can or cannot be composed is difficult. But we try to do the best.  
  

### The override pattern

  
In the [pill 12](http://lethalman.blogspot.it/2014/08/nix-pill-12-inputs-design-pattern.html) we introduced the inputs design pattern. We do not return a derivation picking dependencies directly from the repository, rather we declare the inputs and let the callers pass the necessary arguments.  
  
In our repository we have a set of attributes that import the expressions of the packages and pass these arguments, getting back a derivation. Let's take for example the graphviz attribute:  
```
graphviz = import ./graphviz.nix { inherit mkDerivation gd fontconfig libjpeg bzip2; };

```
If we wanted to produce a derivation of graphviz with a customized gd version, we would have to repeat most of the above plus specifying an alternative gd:  
```
mygraphviz = import ./graphviz.nix {
  inherit mkDerivation fontconfig libjpeg bzip2;
  gd = customgd;
};

```
That's hard to maintain. Using callPackage it would be easier:  
```
mygraphviz = callPackage ./graphviz.nix { gd = customgd; };

```
But we may still be diverging from the original graphviz in the repository.  
  
We would like to avoid specifying the nix expression again, instead reuse the original graphviz attribute in the repository and add our overrides like this:  
```
mygraphviz = graphviz.override { gd = customgd; };

```
The difference is obvious, as well as the advantages of this approach.  
  
Note: that .override is not a "method" in the OO sense as you may think. Nix is a functional language. That .override is simply an attribute of a set.  
  

### The override implementation

  

I remind you, the graphviz attribute in the repository is the derivation returned by the function imported from graphviz.nix. We would like to add a further attribute named "override" to the returned set.

  

Let's start simple by first creating a function "makeOverridable" that takes a function and a set of original arguments to be passed to the function.

  

Contract: the wrapped function must return a set.

  

Let's write a lib.nix:

```
{
  makeOverridable = f: origArgs:
    let
      origRes = f origArgs;
    in
      origRes // { override = newArgs: f (origArgs // newArgs); };
}

```

So makeOverridable takes a function and a set of original arguments. It returns the original returned set, plus a new override attribute.  
This override attribute is a function taking a set of new arguments, and returns the result of the original function called with the original arguments unified with the new arguments. What a mess.  
  
Let's try it with nix-repl:  
```
$ nix-repl
nix-repl> :l lib.nix
Added 1 variables.
nix-repl> f = { a, b }: { result = a+b; }
nix-repl> f { a = 3; b = 5; }
{ result = 8; }
nix-repl> res = makeOverridable f { a = 3; b = 5; }
nix-repl> res
{ override = «lambda»; result = 8; }
nix-repl> res.override { a = 10; }
{ result = 15; }

```

Note that the function f does not return the plain sum but a set, because of the contract. You didn't forget already, did you? :-)  
The variable res is the result of the function call without any override. It's easy to see in the definition of makeOverridable. In addition you can see the new override attribute being a function.  
  
Calling that .override with a set will invoke the original function with the overrides, as expected.  
But: we can't override again! Because the returned set with result 15 does not have an override attribute!  
That's bad, it breaks further compositions.  
  
The solution is simple, the .override function should make the result overridable again:  
```
rec {
  makeOverridable = f: origArgs:
    let
      origRes = f origArgs;
    in
      origRes // { override = newArgs: makeOverridable f (origArgs // newArgs); };
}

```
Please note the rec keyword. It's necessary so that we can refer to makeOverridable from makeOverridable itself.  
  
Now let's try overriding twice:  
```
nix-repl> :l lib.nix
Added 1 variables.
nix-repl> f = { a, b }: { result = a+b; }
nix-repl> res = makeOverridable f { a = 3; b = 5; }
nix-repl> res2 = res.override { a = 10; }
nix-repl> res2
{ override = «lambda»; result = 15; }
nix-repl> res2.override { b = 20; }
{ override = «lambda»; result = 30; }

```
  
Success! The result is 30, as expected because a is overridden to 10 in the first override, and b to 20.  
  
Now it would be nice if callPackage made our derivations overridable. That was the goal of this pill after all. This is an exercise for the reader.

  

### Conclusion

  

The "override" pattern simplifies the way we customize packages starting from an existing set of packages. This opens a world of possibilities about using a central repository like nixpkgs, and defining overrides on our local machine without even modifying the original package.  
  
Dream of a custom isolated nix-shell environment for testing graphviz with a custom gd:  
```
debugVersion (graphviz.override { gd = customgd; })

```
Once a new version of the overridden package comes out in the repository, the customized package will make use of it automatically.  
  
The key in Nix is to find powerful yet simple abstractions in order to let the user customize his environment with highest consistency and lowest maintenance time, by using predefined composable components.  
  

### Next pill

  
...we will talk about Nix search paths. By search path I mean a place in the file system where Nix looks for expressions. You may have wondered, where does that holy <nixpkgs> come from?

  
Pill 15 is available for [reading here](http://lethalman.blogspot.it/2014/09/nix-pill-15-nix-search-paths.html).  
  

To be notified about the new pill, stay tuned on [#NixPills](https://twitter.com/search?src=typd&q=%23NixPills), follow [@lethalman](https://twitter.com/lethalman) or subscribe to the [nixpills rss](http://lethalman.blogspot.com/feeds/posts/default/-/nixpills).