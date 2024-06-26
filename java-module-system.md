# The Java Module System by Nicolai Parlog

## Chapter 1

### The problem: lack of modularity in artifacts/jars - 'JAR HELL'

#### Problem 1: Shadowing

##### Definition

Shadowing occurs when two (or more!) jars have a class with the same FQCN, and the class loader loads **the first one it can find.** The first one loaded in is then always used at run-time and it is said to **shadow** the other classes.

There is no determined way to know in which order the class-loader loads in JARs, so this behaviour can be unpredictable.

##### Example 1: multiple versions of the same library

Say library A has a `com.example.list.Iterator` class which was present in version 1.0.0 of the library but changed in version 2.0.0 of the library. 

If version 1.0.0 was loaded in first, it will use the Iterator present in 1.0.0. And the same goes for loading in version 2.0.0 first...


##### Example 2: shared class with same FQCN in multiple libraries.

if library A has a `com.example.list.Iterator` class and  library B has a `com.example.list.Iterator` class, the first one to be found by the class-loader will be used.


##### A way to overcome it
Build tools already default to using the most recent version by default.

Maven enforcer plugin can enforce not having more than one version:

https://www.baeldung.com/maven-enforcer-plugin#1-ban-duplicate-dependency 

#### problem 2: JARs exist in a 'big ball of mud' in the classpath: there is no concept of encapsulation

- classes in packages deemed for internal use can be easily accessed
- an example of this is `sun.misc.unsafe` used by big projects but intended to only be used internally!

#### problem 3: Misc

- slow startup times because of class loading JAR lookup w/ linear scan
- 'all-or-nothing' for the java platform -> projects that don't need certain java JARs were bundled within the JDK anyway

### The solution: the JPMS (Java Platform Module System)

#### Definition of a module

A module consists of
- a name: _a module descriptor_
- its dependencies upon other modules
- its API (the packages it exports)

This is described in a _module declaration file_ a.k.a. the module-info.java file. Example:

```java
module be.wimdetroyer.foo {
  requires be.wimdetroyer.bar
  exports be.wimdetroyer.api
}
```

#### Lifecycle of the JPMS

##### Step 1 - bootstrapping

The JPMS bootstraps itself. The module system is _code_ so it is defined somewhere. The code is defined in the `java.base` **base module**.
The java module system and the base module bootstrap themselves... :-)

#### Step 2 - Module resolution & verification

The class loader looks up the modules residing in JARs. If a module is found, it is added to the module graph.
As mentioned, modules have **dependencies on other modules**. If the class loader cannot find the module in any JAR, the application will exit with an error.

#### Step 3 - launching the _initial modules'_ main method.

the JPMS launches the main method of the _initial module_ . The initial module is the place where Step 2 began its work. It shouldn't be confused with the base module which is part of the JDK.

The initial module must also contain the psvm method! via the module descriptor it can then be looked up.

#### Step 4 - the JPMS enforcing boundaries at launch time

when module A wants to access a type (class, ...) from module B, the JPMS checks:

- if said type is _public_
- if module B _exports_ that type
- if module A is connected to module B in the module graph (if module A _requires_ module B)

#### Goals of the JPMS

##### Reliable configuration

Dependencies can be found missing at _launch time_ instead of at _run time_ now (see step 4 of the lifecycle of the JPMS).

##### Stronger encapsulation

Only public members of public types of  _exported_ packages of a module are accessible from another module. If it does not satisfy all these constraints, this member is _inaccessible_ (**even when using reflection!**)

> [!NOTE]
>This also applies to the JDK, which, as described in the previous section, was turned
into modules. As a consequence, the module system prevents access to JDK-internal
APIs, meaning packages starting with sun. or com.sun.. Unfortunately, many widely
used frameworks and libraries like Spring, Hibernate, and Mockito use such internal
APIs, so many applications would break on Java 9 if the module system were that strict.
To give developers time to migrate, Java is more lenient: the compiler and JVM have
command-line switches that allow access to internal APIs; and, on Java 9 to 11, run-time
access is allowed by default 


Stronger encapsulation also increases **security** because it forbids access to internal jdk modules.

## Chapter 2 : an example application

This chapter covers an example application that was modularized using the JPMS. sourcecode to be found [here](https://github.com/nipafx/demo-jpms-monitor)
> [!NOTE]
> Easy decoupling can be achieved in java by using functional interfaces, instead of having module B directly having a reference to some property X existing in module A, pass a _supplier_ of that property, removing a hard dependency link!

### Anatomy of module-info

 #### Directives
 Directive are placed inside the module-info.java .
 
 ##### requires
 
 to specify which _modules_ it has dependencies on. (=> pass the _module descriptor_ of the other module)
 
 ##### exports

Specifies the _public API_ of the module. It is a listing of 0 or more _package names_ to which _other modules_ can access the _public types_ from.

### Compiling & running the application


### Chapter 3: defining modules

java module declaration is the module-info.java file, the _descriptor_ is the .class bytecode...
If a JAR is packaged with a module-info.class file or a _module descriptor_ it is said to be a _modular jar_ as opposed to a plain jar.

#### structure of a module-info.java


```
module-name (module descriptor name)
exports {package-name}
requires {other-module-name}
```
##### module-name

make sure the naming of a module somewhat aligns to package naming conventions, even though it doesn't map one to one!

 ##### exports

 No wildcarding exists! This is to make sure exposing what a modules _public API_ is is a _deliberate choice_

#### Types of modules

 - Application module : Typical module you will create in your application
 - Initial module: module where compilation starts with(can from my understanding be an application module)
 - root module: module where JPMS starts initialization (in some (most?) cases this is also the initial module)
 - platform modules (modules within the jdk (java.base, java.sql, ...)
 - incubator modules: experimental JDK modules
 - ...

#### Readability

_Readability_ as a concept is the step that allows for the 'linking' of modules. If module A _requires_ module B, JPMS will let module A _read_ module B.

#### Accessibility: defining public APIs

##### A definition
A type Drink in package `be.wim.foo.bla` existing in the module `be.wim.foo` is _accessible_ to the module `be.wim.bar` if:

-  the type Drink is public
-  the module `be.wim.foo` _exports_ the package `be.wim.foo.bla` as part of its public API, it is in other words: **accessible**
-  module  `be.wim.bar` can _read_ from the module `be.wim.foo`. it _requires_ this module; it is in other words: **readable**

##### Accessibility... stronger encapsulation!

- the public accessor no longer means **public everywhere** but public in this module, and if the module-info exports it, also readable from other modules...
- if a project A pulls in a dependency B which has a transitive dependency C, the module system will **not allow** code in project A that uses code from dependency B, whic in turn uses code from dependency C to be used.
    - transitive dependencies need to be explicitly configured
    - this provides for a safer experience, it forces you to reason about your dependencies!
    - it can be tuned by using _implied readability_

#### Summarizing a module

- Multiple _types_ of modules exist, an exhaustive list is written in the book.
     - Platform modules -> bundled with the JDK
     - application modules -> your own, libraries, ... also comprises the initial module (where compilation starts with) and the root module
 - Modules allow for _reliable configuration_
      - modules present exactly once (unique name)
      - no cycles
      - ...
  - _better encapsulation_
  - Modules turn _plain JARs_ into _modular JARs_


## Chapter 4: building modules


### Structuring a project with modules

#### Default project structure

What I'm most used to, single /src folder for _each_ module. Build tools are tailored for this.

#### 'Established default' proposed by Nicolai

The idea is to have a src/ folder containing your _modules_.

Example structure:

```
/src
├── module.foo (the module directory)
│   ├── module-info.java (module declaration)
│   └── module (a package)
│       └── foo (a package)
└── module.bar
    ├── module-info.java
    └── module
        └── foo
```

Tests would be under a /test-src folder and contain tests for the respective modules.


Skipping the rest of the chapter as I don't seem this low-level knowledge relevant for me.

# Chapter 5

Skipped.

#  Chapter 6 - problems when moving to java 9 

Java is meant to be backwards compatible, but the ecosystem can use behaviour which is _unsafe_ (read: undocumented, unstandardized, deprecated, ...). and at that point, it can pose some issues...

Examples:
- JEE modules are apparently special, more on that later...
- using internal APIs
- split packages
- ...

**The older your project, the more likely it might be a hassle to upgrade**

> [!NOTE]
> To me, it doesn't seem something which should affect me _too_ much... lucky me, I guess ;-)



Stoppe reading at 6.1.1 Why are the JEE modules special?

**
