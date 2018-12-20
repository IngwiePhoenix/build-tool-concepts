# Build Tool Concepts
Within this document, I would like to look at possible concepts for a build tool. This is due to the fact that I want to create a build tool, but am stuck at a variety of questions and implementation-speciric details.

## Why `make` a build tool?
No pun intended here but Make is actually a great tool. It and Ninja have one specific thing in common: They are actually not limited to C and C++, but you could easily create build descriptions for JavaScript, Go and other languages without much of an issue. But something else they have in commin is... well, they are "stupid". Neither of the two really knows the difference between a shell command like `rm` or a compiler such as `gcc`. Let alone `CL.EXE` or `LINK.EXE` on Windows... And as they don't know that, you can not do, what I like to call, "configuration". For instance, when you develop cross-platform applications, you are likely going to have to either include `unistd.h` or `windows.h` down in your code _at some point_. But did you know that there is a difference on OS X and Linux in regards to `malloc.h`? And what about extremely special use cases like Emscripten, Cheerp or WebAssembly, where the given "standard library" is a *lot* different than your regular C standard library?

As `make` really only does work it's way through tasks, you are let alone on the configuration part. And this is something I personally see super essential on what a build tool schould be able to do: It should be able to help you, without interfering.

## Configure scripts vs. `#ifdef`-Macros
Consider my example from above regarding `malloc.h`. You could either do:

```
#if defined(WIN32) || defined(_WIN32) || defined(MSVC)
#include <windows.h>
#else
#ifdef __APPLE__
#include <malloc/malloc.h>
#else
#include <malloc.h>
#endif
#endif
```

...or you could write a dedicated `config.h` file which includes ALL of such checks and includes the required headers as you go. But do you know what makes this harder? Eyup, dependency management :).

Consider having two projects, both of which are part of a bigger one. Say you have an audio player UI frontend, and a corresponding backend. These two will need a lot of different and yet partially shared dependencies. The backend will want to know how to best implement codecs and decoders, while the frontend will want to know all about the UI library in use. But both of them will want to do logging and memory management. So you may end up with `config/shared.h`, `config/frontend.h` and `config/backend.h`. And if you maintain this alone, this will be okay-ish and maintainable. But what if you have a team and they ALL want to modify either of these three? You will easily run into some super clogged header files there. And although I am not much into static analyzing of code, I am highly doubting that this is an aproach those analyzers would like alltogether...

Another method would be to have a configuration script. But... how to make one? You may have written one in a previous project, based on Autotools, and you decide to reuse that in a new project, only to realize that you somehow managed to have a different version of `autoconf` installed than you had back then, and also notice that some of your macros have turned up results that...look simply weird. So what will you do? Well... you will re-write most of that script, to match your new version, now. But at the same time, you will lock this project up into a UNIX environment - because Autotools relies on GNU tools, which mostly run on a shell like `bash`, which is not the default in Windows. Let alone, that now you would have to tell other developers to install the whole of MinGW or Cygwin first in order to even issue a single build command! This basically inflates your project by the size of either MinGW or Cygwin during development on Windows. Congratulations!

There is a method, that I have discovered - and sort-of maintained very, very losely - was to use a C-based build tool within C/C++ application projects. It's called [cDetect](https://github.com/IngwiePhoenix/cDetect) and only needs a compiler to run - no shell, but a compiler. Therefore, any compiler. I bet it would even run on TCC flalessly, really. So what does this mean? Well, if you think in Ninja (which, if I have to hand-craft a build-description, is my go-to choice):

```ninja
rule CC
  command = $CC -c $in -o $out
rule EXE
  command = $CC $in -o $out
rule CMD
  command = [ -f "./$in" ] && ./$in || $in

build configure: EXE configure.c
build config.h: CMD configure
build myapp.o: CC myapp.c
  deps = config.h
build myapp: EXE myapp.o
```

Now, sure, I kind of forgot how to properly notate dependencies in Ninja in regards to files that are not directly meant for a specific command to be executed. But you can tell that this file will ONLY work on a shell like Bash and on compilers using `-c` and `-o` flags, like GCC or Clang. Clang-CL might work with this, but I haven't used that one yet.

But what you can see though, is that cDetect is used to create the desired `config.h` file, so we can at least pick up details about headers and types that we need - but Ninja still has no idea what a compiler is - we are just assuming that using `-c` and `-o` will _totally work as expected_...which it will not, on Windows.

You could use this cDetect approach with something like GYP and have yourself a Visual Studio solution made off the build description, but I am not sure how well this would turn out. Alternatively, you could use CMake.

## DSLs
And by mentioning CMake, I am going to let loose on why I feel like CMake has ruined build tools for me: Its horrible, horrible, horrible "language".

Usually, when you write a program in C, C++ or PHP or even JavaScript, you will be used to a specific kind of syntax already. In all these examples, this would be a C-ish syntax:

```
# Variables:
[type | "var"] varName = expr;

# Functions:
[returnType | "function"] funcName ( [type?] expr , ... ) ;
```

This is the _most basic_ way of understanding C, JavaScript and, to a degree, PHP. However, you have to add a dollar-sign before a `varName` to make it a proper definition... like `$foo = "bar";`.

Now, what does CMake look like? Well... anything BUT like C. Sure, it uses paranthesis, but where are the commas in parameter lists? And, what, in a "function call" is a label and an actual argument? Just look at `ExternalProject_Add` in the CMake documentation to get what I mean. Also, I highly miss the assignment operator (`=`) which tells me, as a programmer, that the right-hand value is assigned to the left-hand variable. Take that short foo-example from before: The string "bar" is assigned to `$foo`. But it really gets funny when you work with lists (arrays) and maps (objects). This stuff will quickly look like a mess... a big mess.

So, why aren't we leveraring a syntax that we already know? Over there in the JavaScript world, there are _a lot_ of build tools. I can name a few without looking at NPM: Gulp, Grunt, Parcel, WebPack, Rollup - heck, even NPM itself could be used as one! But the great thing about most of them is, that they don't need to implement a new language - they will just use what is already given, and try to _mimic_ existing APIs. Like, in Gulp, you use `.pipe()` a lot, which is basically this on a command line: `cat test.js | terser > out.js`. In Grunt, this would probably be something like: `Gulp.src("./test.js").pipe(terser()).pipe(fs.createWriteStream("./out.js"))`.

But where Gulp fails is where NPM is flawed: You can not use Gulp without downloading _the whole internet_... I'm talking about dependencies.

```
Ingwie@Ingwies-Macbook-Pro.local ~/W/I/build-tool-concepts $ npm i -g gulp
npm http fetch GET 200 http://registry.npmjs.org/gulp 90ms
npm http fetch GET 200 http://registry.npmjs.org/gulp/-/gulp-4.0.0.tgz 37ms
npm http fetch GET 200 http://registry.npmjs.org/glob-watcher 35ms
npm http fetch GET 200 http://registry.npmjs.org/undertaker 65ms
npm http fetch GET 200 http://registry.npmjs.org/gulp-cli 66ms
npm http fetch GET 200 http://registry.npmjs.org/vinyl-fs 70ms
npm http fetch GET 200 http://registry.npmjs.org/async-done 24ms
npm http fetch GET 200 http://registry.npmjs.org/is-negated-glob 28ms
(... snip ...)
+ gulp@4.0.0
added 318 packages from 217 contributors in 16.052s
```

Now, I have a downstream of 400 megabit per second and by the time, a very well filled NPM cache. But new developers may not have either of those - let alone, they may not even have Node installed. So first, they have to go to the NodeJS website, download node, install node, then type in some NPM commands to set up a project and to finally install Gulp...and grab themselves a cup of coffee. This is going to take a while.

So although Gulp does many things right, it is flawed by the same issue that a linux-only build on Windows is flawed with: You *have to* download a huge amount of extra stuff and configure that first, before you can do *anything*.

## The chicken/egg problem.
Who came first? The compiler, or the OS? Well, you know the answer. But in the case of a build tool, this is about as important as it gets: Do you install your toolchain first and THEN the build tool, or do you first download the build tool, to build a toolchain?

At first, this may not seem that complex. You'd just download your MSVC or XCode, and then download any of the build tools you want, and have them pick up your toolchain. But, what about cross-compiling? This is not a very common case, true, however you might need to cross-compile at some point. So... in this case, the build tool needs to be available, before the toolchain compilation. But you will also need a compiler to compile the bootstrap of your new toolchain. Oh, what a joy!

In most cases, you can just download yourself a build tool as a pre-compiled binary, and have your native toolchain ready to go and installed, and now you can just build your toolchain. Well - this is the most common case. But what if the toolchain comes with it's own requirements and dependencies? You will need to have to install those first. For this, you may use a package manager (which you *also* need to install) and a few helpers to set up your environment variables properly. Then, go through the dependency installation or building, and then come back to that toolchain. If you are lucky, the versions specified for it's dependencies were well picked and everything ill work just fine. But if not, then you are in for some really crazy ride. :)

So - is there a chicken/egg problem? Absolutely. A lot of those, when dependency management comes into play, and even more if you wish to cross-compile.

But why do I mention this? Because, there should be ways to kill this problem alltogether. I mean, we have so many package managers like NPM, Homebrew, CPAN, Composer, Ruby Gems and what not - why is there none that is dedicated to the same level that these examples are, in a more general purpose way? What I mean is, we can use NPM to download all of a NodeJS' applications dependencies, even use semantic versioning to track those and set up many scripts that run during the installation withon `node_modules`. But this does not seem to really exist for C and C++... Currently, there are these ways of tracking dependencies:

- Copy your dependency into your source tree.
- Use Git submodules.
- Use native package managers like APT, Homebrew or NuGet (feel free to correct me on that one, never used this!)
- Use shell scripts to download dependencies.
- Use awkward package managers, written _on top_ of build tools like CMake.

As you see: there is no uniform way. And yet, we _love_ our `package.json`-ish files a lot...and yet, we don't have a tool to grab such dependencies for us? I mean, you could make a couple of package description files, and sequentially call the various package managers to download all your lovely dependencies... But this _will_ end up bad.

Think about this: When you see a C++ project next time, look at how it stores it's dependencies. Just do that, and you will notice that it is handled slightly different everywhere.

## Modularity
How do we usually look at modules? Either...

```java
package me.ingwie.examples.javasnippet;
public class Snippet {
  public static void foo() {}
}

// Later --> import me.ingwie.examples.javasnippet.Snippet;
```

Or...
```js
// @file: src/example/foo.mjs
export function fooSpeaker() {}

// Later --> import {fooSpeaker} from "example/foo";
```

Or...
```c++
namespace IngwiePhoenix {
  namespace CXXExample {
    void foo() {}
  }
}

// Later --> use IngwiePhoenix::CXXExample::foo;
```

Or...
```php
namespace IngwiePhoenix\PHPExample;
function foo() {}

// Later --> use \IngwiePhoenix\PHPExample\foo;
```

There are many, many more examples. But how do we do that in CMake or other build tools? Uh... well, most people seem to subsequently include their module's `CMakeLists.txt` files and build little static library targets off them, to have them then linked into one big thing. And, I am very certain, this is not the only ay people do this. Even more so, this is just one example out of many more from other tools. So, just like with the DSL, I ask you: Why don't we do it, like we do it in code? For instance: Grouping your targets in subsequent namespaces would allow somebody else to easily reuse your code. Like, in JavaScript, you may not always import the whole module, but just a fraction instead, if you only need that very part. Like, let's say: `import {default as consoleReporter} from "ATestingFramework/reporters/console";`. You may only want this reporter for your application, because you like it and it's API a lot. Unfortunately, this is a little bit harder with stuff like C++... In fact, I have no good example for this at hand, since I have never really seen a real modular build like this. It appears, though, that you can link against LLVM in a modular fashion. But, this is the only example I really know of.

So, why does no build tool support something like namespaces? I... have no idea why.

## More than one universe.
Okay, so I have mostly mentioned CMake, GYP, Make, Autotools, Grunt and a few more. But those are, obviously, not the only ones. There are systems like Gradle, Bazel, Buck, Pants, Ant, Maven, ... You name it. But what they mostly have in common is - except for a few exceptions - that they are bound to one, and only one universe. You can make some of those tools also work on things that are not exactly from their "universe", but you will not have an easy task with that. Make and Ninja are very much an exception here - they don't really belong to a universe at all, and you can use them for whatever you want. You remember that Gulp example from above? You could easily implement that with Ninja or Make as well - no problem. But other than that... you are very stuck to the respective "universe" that a build tool belongs to. And this means that:

- You may need multiple build tools in one project, especially in large ones.
- You may have to hack together scripts and methods to work with specific files - say, to optimize images.
- You may find yourself dabbling in an area of your respective tool, that is actually not recommended, and yet you may need that.

In other words: There is a huge number of issues you may run into. Thus, each of those build tools will require you to actually do something, before you can use them.

# Concept: IceTea
First and foremost: I have been doing this research-y stuff about build tools and build system since _years_. It all started back when I was excited about AppJS, and wanted to later build a similiar framework called "Phoenix Engine". But, Electron happened instead, and basically blew away my motivation. Then, there is also this huge community project I am planning called "BIRD3", which will use a very huge NodeJS based backend, but I want to utilize stuff such as gRPC, which requires a protocol compiler to be used. So this will bring me to some issues in the future for sure.

This is why I decided to create IceTea. There is a sort-of working version, utilizing [ObjectScript](https://github.com/unitpoint/objectscript), that does some basic stuff. But, ObjectScript is not actively maintained anymore, as it seems, and for a tool that needs to go with time, this may lead to some issues... Thus, I myself have zero knowledge about how to write my own scripting language, let alone a VM to execute it in.

But although I am still researching, here are some features I would like to create.

## Namespaces, namespaces and even more namespaces!
Namespaces can appear in more than one way - either as part of a project structure, or as part of the program's syntax, or even as part of the various packages published to something like NPM. Now, in IceTea, I want to utilize namespaces for many, many reasons.

First, here is some concept code:
```
namespace("IngwiePhoenix/Examples") {
  target("foo") {

  }
}
```

What you can see here is quite basically the description of a namespace and an object residing within it. You could translate this into a tree like this:
```
IngwiePhoenix
| Examples
| | foo
```

Or, maybe into something like an XPath: `IngwiePhoenix > Examples > foo`.

Or, XML?
```xml
<namespace name="IngwiePhoenix">
  <namespace name="Examples">
    <target name="foo"/>
  </namespace>
</namespace>
```

The idea is, to be able to structure everything into namespaces. Those may even be completely in dis-connection to a project's file structure on a whole good purpose:

- Abstract your targets (binaries, libraries, ...) into objects.
- Organize objects in namespaces.
- Point to objects in another namespace, without knowing the original project's layout.
- Provide modules on a slightly altered layout.

For instance, take this as an example:
```
import cURL from "curl";
import mbedtls from "mbedtls";

namespace("IngwiePhoenix/Examples") {
  target("mini-downloader") {
    configure: function() {
      // Add cURL into your project in a way that you can configure curl yourself.
      var myCurl = cURL.extendThis(this);

      // Chose a different SSL library
      myCurl.ssllib = mbedtls;

      // Disable FTP
      myCurl.ftp = false;

      // Let cURL configure itself and generate all files needed,
      // and add them to your project.
      return myCurl.configure();
    },

    needs: [
      // You may want to use mbedtls' crypto library, so you can depend on
      // it. Your compiler flags and settings would be updated to add the
      // required include directories, link flags, and alike.
      "mbedtls/crypto",

      // But since you have a module object, you may use that instead:
      mbedtls.getTarget("crypto")
    ]
  }
}
```

Right here, we are adding cURL and mbedtls to our project _without even knowing what they look like_. We don't know the file structure, and nor do we really know what flags we would have to add to compile against them. Heck - we may not even know if the packages have bundled binary distributions with them or not.

You can see that namespaces are used to allocate build objects, while a regular module may be used to provide additional functions. This can reach from small helpers for working with dependencies, up to whole workflows and toolchains that do other totally cool things. Therefore: The namespace is used for describing build related data, while the rest is used to write proper build scripts.

## Types
A build tool should always have those two types, at any time:
- Target
- Action

A target refers to something that can be built, and an action defines what targets should be built. IceTea extends this a little by adding these types:
- Namespace
- Step
- Rule
- Workflow
- Platform
- Toolchain
- Tool

I will explain them a little bit more. Well, obviously I will skip namespaces here, since I elaborated on them already.

### Step and Rule
A step represents going from one part of the build, to another. Like compiling a `.c` file into a `.o` file. And a rule defines what happens when all steps have been dune - basically, the last command.

```
      step       rule
main.c -> main.o -+
lib.c  -> lib.o  ||
                 vv
                 main.exe
```

First, the steps for "stepping from `.c` to `.o`" are performed, then the rule for "turning all `.o` files into an `.exe` file" is performed.

You may even chain multiple rules. For instance, if you have a MacOS application, you may have binaries, libraries and frameworks, that all need to go into the same application bundle. In this case, the chain would go almost as to what you see above, and extend just a little beyond to turn the built files into a bundle.

```
main.c -> main.o => main.exe

lib.c -> lib.o => libutils.a

frm.m -> frm.o => libMyFramework.dylib => MyFramework.framework/MyFramework

[
  main.exe
  libutils.a
  MyFramework.framework
] => MyApp.app
```

In this example, we built a binary, a static library and a framework off a shared library and finalized this into a single application bundle.

### Workflow
Of course - even IceTea is rather "stupid". You have to tell it what a workflow may look like - as in, working from one end to another of a chain. Here is an example for the Framework workflow:

```
.c -> .o
*.o -> .dylib
.dylib -> .Framework/
```

This translates into:
1. Compile ONE C source into ONE object.
2. Link ALL objects into ONE shared library.
3. Create a Framework folder and copy ONE shared library into it.

Obviously, we could even add copying headers and alike, which would be no problem at all.

But how about non-C stuff? Well, a workflow is basically just an array consisting of multiple routes. So, you can even extend those! Take for instance [gccx](https://github.com/mbasso/gccx) and assume that we are using the Emscripten toolchain instead of our native compilers. The default for compiling C++ into a binary would be:

```
.cpp -> .o
*.o -> .exe
```

Now, how about we prepend `.cpx` files? We'd just create a new workflow that starts with `.cpx` files, and hand the resulting `.cpp` file over to the regular workflow.

```
workflow 1:
  .cpp -> .bco
  *.bco -> .js

Workflow 2:
  .cpx -> .cpp
  (workflow 1)
```

This would result in such commands:
```
gccx main.cpx -o main.cpp
em++ -c main.cpp -o main.bco
em++ main.bco -o main.js
```

This way, we could even streamline lexer and parser generators like Byacc or re2c into our source, and just add new workflows for the new file types that we want to support.

### Platform
A platform is, as to put it in perspective to cross-compiling, the *host platform*. Those platform objects may allow you to skip through configuration, as they alredy know details about the platform, and can therefore save you some time - or even offer you specific helpers, adapted workflows, special targets, and even more, that a platform supports out of the box. For instance, a platform may export a method called `numThreads()`, to allow you to pick up on how many threads can be utilized, while in reality, this method will run a shell command specific to each individual platform. However - Platforms are actually classes, and you may even extend it and use a different one.

The default platform is leant against target triplets: `unknown-unknown-unknown`. Ergo: This is a dumb platform, it knows absolutely nothing, and will figure everything out as you go. But meanwhile, there may be a platform called: `x86_64-apple-darwin`. This is MacOS, and this platform may already know the locations of compilers and libraries, which ones are available and which ones are not - and other shortcuts that could make configuring much faster, since some things are already known, and dont need to be detected again.

But what if you wanted to cross-compile, or use a specially enhanced platform class? My idea is, that you may set this *in the root build description*. So there should be a function that allows you to pick which platform should be used. Mind you - cross-compiling and tooling and other stuff, should happen in a...

### Toolchain
A toolchain contains information about how to do something specific. The default, "Native" toolchain, for instance, will know how to compile C, C++ and Assembly into objects, libraries and binaries.

BUT the MacOS platform may also provide parts that can be used in a toolchain. For instance: It knows how to build an application bundle. This is why a toolchain typically exports rules, steps and workflows with clear identifiers. The step for `.c -> .o` may in fact be called `CC`. The rule to build an executable may be called `EXE` or `LINK`. A platform may even utilize that and create new rules, based on existing steps, so that a platform's super-special targets - like making application or framework bundles - do not need to fill up a toolchain. However, this may also happen vice-versa, where a toolchain checks which platform is in use, and disables some steps right away, if they would not be compatible.

But it gets much more interesting when you think about cross-compiling. Here is an example:
```
import VitaTC from "toolchains/arm-vita-ebai";

target("myapp", "EXE") {
  src: glob("./src/*.cpp")
}

IceTea.setToolchain(VitaTC);
```

As you can see, we imported a special toolchain for the Playstation Vita, and just added one call - and that's it. The now used toolchain will effectively only build for the PlayStation Vita. But, the true magic is yet to happen. So keep this example in mind for later.

This method may be used for all sorts of toolchains. You may extend the original native toolchain and extend it by some rules or targets - or you directly mix other tools within. This is especially useful when you have some general purpose tools that you wish to add to all toolchains, at all time.

### Tool
Frankly speaking: This may wrap a function or command into a class, to allow it to be used in an abstracted way. So, here is a most-basic example:

```
tool("bash-script") {
  init: function() {
    // Check if and where bash is installed on this system...
    this.bashLocation = ...;
  },
  run: function(args) {
    var cmd = ShellCommand();
    // Set the command to run
    cmd.program = this.bashLocation;
    // Apply arguments
    cmd.args = args;
    // Run the command and return it's exit code.
    return cmd.run();
  }
}
```

Now, there are two ways of using a tool. You can use it within your own code by searching for it:
```
var bashScript = tool.get("bash-script");
bashScript("./something", "really", "fancy");
```

Or, you can call it from the command line:
```
$ icetea -T bash-script -- ./something really fancy
```

Or you can even link yourself an executable for convenience:
```
$ ln -s (which icetea) ./bash-script
$ ./bash-script ./something really fancy
```

Tools are useful especially in targets, steps and rules.

# Dependencies
IceTea will utilize it's own idea of a package manager - which will, however, obtain packages from not a centralized server, but from wherever you want it to. And - packages are not just libraries, tools or such... They may very well be platforms, or...entire toolchains.

Now, imagine this: You want to cross-compile your project to Linux from your Windows, and you have no idea how to set up such a toolchain - or, are too lazy to do so. In IceTea, you may add something like this into a `package.json`:

```json
{
  "devDependencies": {
    "github.com/icetea/tc_arm-musl-linux3.6": "#latest"
  }
}
```

Now, when you try to perform your build, it will go through as usual, but before it does, the package manager will be invoked to download this very dependency. This may very well mean, that a pre-compiled, ready-to-use, freestanding cross-compiler is downloaded into the dependencies folder. And all you need to do now is to tell IceTea to use this - which you can do from the command line too!

```
$ icetea --tc=arm-musl-linux3.6
```

And...done. The toolchain will override the native one, replace a variety of compilation rules, and even replace feature detectors so that they use this compiler now.

But... this is not all that you can do. Hey, this is a modular build tool - how about you deploy that build to Bintray? Or wrap up multiple builds into a zip file and upload that to your remote server, call a RESTful API to have that archive be made visible on your website? All of this is no problem - In fact...

```
import ITPM from "itpm";
import easyCurl from "curl/easycurl";
action("build-many") {|do|
  var packages = ITPM.getPackages().filter({|p| "toolchain" in p.type})
  packages.each({|pkg| do({
    toolchain: pkg.name,
    phases: ["build", "dist"],
    profile: "release"
  }) });

  var uploader = easyCurl({
    method: "POST",
    host: "..."
  });
  uploader.addFile(fs.getDirList(__DIST_DIR__));
  uploader.run();
}
```

Done! \o

## Feature detection
I have shown you thus far, what IceTea is capable of doing - or, well, what it should be capable of doing, when I get to actually write it... One thing I have not talked about in a while is feature detection. Now, this is super straight forward:

```
var check = Detect("CC");
check.header("stdio.h");
check.lib("m");
check.sizeof("void *");
check.functionInHeader("printf", "stdio.h");
check.typeInHeader("uint_t", "stdint.h");
check.symbolInLib("printf", "c"); // Dont worry - the proper library name is generated for you! :)

check = Detect("CXX");
check.classInHeader("std::string", "iostream");
check.ifInstantiable("std::string", "new std::string('c')", {
  header: "iostream"
});

check = Detect("PHP");
check.module("pcre");
check.zts();

check = Detect("NodeJS");
check.version();
check.globalModule("npm");
```

This is what a target's `configure` method is all about - checking, and configuring. That way, each project can query the system for things that it needs to know before processing any further!

## Where is this? I want to try it!
...unfortunately, you can't. I am currently stuck on finding a scripting language that has the syntactic shugar that I need in order to create a nice, mostly delcarative syntax for a build description.

Here is a little bit of what I found, in a table, and a legend to it. Enjoy :)

### Features wanted:
- Block arguments
  - This: `f(a) { ... }` should call "f" with the arguments "a" and a _block_.
  - _block_ is either a callback function ala `{|args, ...| stmts... }`
  - or an object ala `{key: "value"}`.
- Error handling
- Classes and class inheritance
- Globals: In order to share state, having a global environment would make things easier, as well as defining some helper functions and generic macros.
- Module loading: I need to be able to structure the entire backend of IceTea. Therefore, I definitively need module loading - but also to allow others to make cool stuff for it, too! :)
- One-line compile: I need to be able to compile this language _really fast_. The most optimal way to do so is to issue as few commands as possible.

### Scripting languages

* [*ObjectScript*](https://github.com/unitpoint/objectscript)
  - Block arguments: Yes, sort of.
    - Return a function from a function to abuse it's rule of "no paranthesis required when passing only one argument"
    - So this: `var f = function(a) {return function(b) {print(a); print(b); }}`
    - Could be called like: `f("a") "b"` and print `a b`.
  - Errors: Try, Catch, Exceptions.
  - Classes: It is a prototype based language and has no dedicated class syntax. But, kinda has them still.
  - Globals: Absolutely. `_G` is an object representing the global scope.
  - Module loading: You can litterally implement it yourself by using `compileScript()`.
  - One-line compile: `g++ os.cpp src/*.cpp -I src -o os` - done.
- [*Wren*](http://wren.io)
  - Block arguments: Yes - no hacks required.
  - Errors: You can abort a fiber to kinda throw an error.
  - Classes: It is completely class-based - but it's constructors are quite a gem.
    - Unfortunately, you will NEVER see something like `foo()`... It's class-system goes as far as there are not being any global functions, at all. So, if you want to call a function, you basically call it's `.call()` method. So... You do `foo.call()` instead.
    - Also: No dedicated global functions... *at all*.
  - Globals: Nope. Each module to it's own, no shared global scope.
  - Module loading: Yes.
    - Only allows importing capital-letter-first classes, like `Foo`.
  - One-line compile: Uh... sort of. You can't build a binary on one line, unless you have libuv pre-installed. But to build a `libwren`, you can do this:
    - `gcc -c -I src -I src/include -I src/optional -I src/vm src/vm/*.c src/optional/*.c`
    - `ar rcu libwren.a *.o`
- [*Gravity*](http://gravity-lang.org/)
  - Block arguments: No.
  - Errors: Abort a fiber to cause an error.
  - Classes: Yes, even classes-in-classes.
  - Globals: I am not sure...
  - Module loading: Not implemented (yet!).
  - One-line compile: Yes - but, I recommend using environment variables...
    - `CFLAGS="-I src/compiler/ -I src/optionals/ -I src/runtime/ -I src/shared/ -I src/utils/"`
    - `gcc $CFLAGS src/compiler/*.c src/optionals/*.c src/runtime/*.c src/shared/*.c src/utils/*.c src/cli/gravity.c -o gravity`

# Having anything to say?
If you wish to talk about this, have suggestions or ideas, feel free to mail me on my public email adress listed in my profile.

Thanks for reading! :)
