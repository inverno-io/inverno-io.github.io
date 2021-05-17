$properties(base = ../../../../, title = Winter Reference Guide)

[winter-getting-started]: ${base}docs/getting-started/html/index.html
[winter-javadoc]: ${base}docs/release/api/index.html

[java-module-system]: https://en.wikipedia.org/wiki/Java_Platform_Module_System
[inversion-of-control]: https://en.wikipedia.org/wiki/Inversion_of_control
[dependency-injection]: https://en.wikipedia.org/wiki/Dependency_injection
[apache-license]: https://www.apache.org/licenses/LICENSE-2.0
[github-issue]: https://github.com/winterframework-io/winter/issues

<div class="heading"> 
	<h1 class="heading-title">Winter Reference Guide</h1> 
	<p class="heading-subtitle">Version: 1.0.0</p> 
	<p class="heading-subtitle">Author: <a href="mailto:jeremy.kuhn@winterframework.io">Jeremy Kuhn</a></p>
	<a class="btn btn-primary d-none d-lg-inline-block d-print-none m-5 position-absolute bottom-0 end-0" href="../reference.pdf" role="button" download="winter-reference-guide-1.0.0.pdf"><i class="bi bi-download"></i> Reference guide.pdf</a>
</div>

$toc( level=3 )

$doc

# Introduction

The **Winter Framework** has been created with the objective of facilitating the creation of Java enterprise applications with maximum modularity, performance, maintainability and customizability. 

New technologies are emerging all the time questioning what has been working for years, We strongly believe that we must instead recognize and preserve proven solutions and only provide what is missing or change what is no longer in line with widely accepted evolutions. The Java platform has proven to be resilient to change and offers features that make it an ideal choice to create durable and efficient applications in complex technical and organizational environments which is precisely what is expected in an enterprise world. The Winter Framework is a fully integrated suite of modules built for the Java platform that fully embrace its philosophy by keeping things well organized, strict and explicit with clean APIs and comprehensive documentation.

The Winter framework is open source and licensed under version 2.0 of the [Apache License][apache-license].

## Design principles

A Winter application is inherently modular, **modularity** is a key design principle which guarantees a proper separation of concerns providing flexibility, maintainability, stability and ease of development regardless of the lifespan of an application or the number of people involved to develop it. A Winter module is built as a standard Java module extending the [Java module system][java-module-system] with [Inversion of Control][inversion-of-control] and [Dependency Injection][dependency-injection] performed at compile time.

The Winter Framework extends the Java compiler to generate code at compile time when it makes sense to do so which is strictly why annotations were initially created for. When done appropriately, **code generation** can be extremely valuable: issues can be detected ahead of time by analyzing the code during compilation, runtime footprint can be reduced by transferring costly processing like IoC/DI to the compiler improving runtime performance at the same time.

The framework uses a state of the art threading model and it has been designed from the ground up to be fully non-blocking and reactive in order to deliver very **high performance** while simplifying development of highly distributed applications requiring back pressure management.

The inherent modularity of the framework based on the Java module system guarantees a nice and clean project structure which prevents misuse and abuse by clearly separating the concerns and exposing **well designed APIs**.

Special attention has been paid to **configuration** and **customization** which are often overlooked and yet vital to create applications that can adapt to any environment or context.

## Getting help

We provide here a reference guide that starts by an overview of the Winter core, modules and tools projects which gives a good idea of what can be done with the framework followed by a more comprehensive documentation that should guide you in the creation of a Winter project using the Winter distribution, the use of the core IoC/DI framework, the various modules including the configuration and the Web server modules and the tools to run, package and distribute Winter components and applications.

The [API documentation][winter-javadoc] provides plenty of details on how to use the various APIs. The [getting started guide][winter-getting-started] is also a good starting point to get into it.

Feel free to report bugs and feature requests or simply ask questions using [GitHub][github-issue]'s issue tracking system if you ran in any issue or wish to see some new functionalities implemented in the framework.

# Overview

$include( source=${WINTER_WORKSPACE}/winter-core/README.md, end=121, heading-offset=1)
$include( source=${WINTER_WORKSPACE}/winter-mods/README.md, end=134, heading-offset=1 )
$include( source=${WINTER_WORKSPACE}/winter-tools/README.md, end=22, heading-offset=1 )
$include( source=${WINTER_WORKSPACE}/winter-dist/README.md, end=312 )
$include( source=${WINTER_WORKSPACE}/winter-core/doc/reference-guide.md, resources = img )
$include( source=${WINTER_WORKSPACE}/winter-mods/doc/reference-guide.md, resources = img )
$include( source=${WINTER_WORKSPACE}/winter-tools/winter-maven-plugin/README.md )
$include( source=${WINTER_WORKSPACE}/winter-oss-parent/README.md )