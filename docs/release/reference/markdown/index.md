$properties(base = ../../../../, title = Reference Documentation)

[inverno-getting-started]: ${base}docs/getting-started/html/index.html
[inverno-javadoc]: ${base}docs/release/api/index.html

[java-module-system]: https://en.wikipedia.org/wiki/Java_Platform_Module_System
[inversion-of-control]: https://en.wikipedia.org/wiki/Inversion_of_control
[dependency-injection]: https://en.wikipedia.org/wiki/Dependency_injection
[apache-license]: https://www.apache.org/licenses/LICENSE-2.0
[github-issue]: https://github.com/inverno-io/inverno-core/issues

<div class="heading"> 
	<h1 class="heading-title">Inverno Framework Documentation</h1> 
	<p class="heading-subtitle">Version: ${VERSION_INVERNO_DIST}</p> 
	<p class="heading-subtitle">Author: <a href="mailto:jeremy.kuhn@inverno.io">Jeremy Kuhn</a></p>
	<a class="btn btn-primary d-none d-lg-inline-block d-print-none m-5 position-absolute bottom-0 end-0" href="../reference.pdf" role="button" download="inverno-framework-documentation-${VERSION_INVERNO_DIST}.pdf"><i class="bi bi-download"></i> Inverno Documentation.pdf</a>
</div>

$toc( level=5 )

$doc

## Introduction

The **Inverno Framework** has been created with the objective of facilitating the creation of Java enterprise applications with maximum modularity, performance, maintainability and customizability. 

New technologies are emerging all the time questioning what has been working for years, We strongly believe that we must instead recognize and preserve proven solutions and only provide what is missing or change what is no longer in line with widely accepted evolutions. The Java platform has proven to be resilient to change and offers features that make it an ideal choice to create durable and efficient applications in complex technical and organizational environments which is precisely what is expected in an enterprise world. The Inverno Framework is a fully integrated suite of modules built for the Java platform that fully embrace this philosophy by keeping things well organized, strict and explicit with clean APIs and comprehensive documentation.

The Inverno framework is open source and licensed under version 2.0 of the [Apache License][apache-license].

### Design principles

A Inverno application is inherently modular, **modularity** is a key design principle which guarantees a proper separation of concerns providing flexibility, maintainability, stability and ease of development regardless of the lifespan of an application or the number of people involved to develop it. A Inverno module is built as a standard Java module extending the [Java module system][java-module-system] with [Inversion of Control][inversion-of-control] and [Dependency Injection][dependency-injection] performed at compile time.

The Inverno Framework extends the Java compiler to generate code at compile time when it makes sense to do so which is strictly why annotations were initially created for. When done appropriately, **code generation** can be extremely valuable: issues can be detected ahead of time by analyzing the code during compilation, runtime footprint can be reduced by transferring costly processing like IoC/DI to the compiler improving runtime performance at the same time.

The framework uses a state of the art threading model and it has been designed from the ground up to be fully non-blocking and reactive in order to deliver very **high performance** while simplifying development of highly distributed applications requiring back pressure management.

The inherent modularity of the framework based on the Java module system guarantees a nice and clean project structure which prevents misuse and abuse by clearly separating the concerns and exposing **well designed APIs**.

Special attention has been paid to **configuration** and **customization** which are often overlooked and yet vital to create applications that can adapt to any environment or context.

### Getting help

We provide here a reference guide that starts by an overview of the Inverno core, modules and tools projects which gives a good idea of what can be done with the framework followed by a more comprehensive documentation that should guide you in the creation of an Inverno project using the Inverno distribution, the use of the core IoC/DI framework, the various modules including the configuration and the Web server modules and the tools to run, package and distribute Inverno components and applications.

The [API documentation][inverno-javadoc] provides plenty of details on how to use the various APIs. The [getting started guide][inverno-getting-started] is also a good starting point to get into it.

Feel free to report bugs and feature requests or simply ask questions using [GitHub][github-issue]'s issue tracking system if you ran in any issue or wish to see some new functionalities implemented in the framework.

## Overview

$include( source=${INVERNO_WORKSPACE}/inverno-core/README.md, end=124, heading-offset=2 )
$include( source=${INVERNO_WORKSPACE}/inverno-mods/README.md, end=335, heading-offset=2 )
$include( source=${INVERNO_WORKSPACE}/inverno-tools/README.md, end=35, heading-offset=2 )
$include( source=${INVERNO_WORKSPACE}/inverno-dist/README.md, end=321, heading-offset=1 )
$include( source=${INVERNO_WORKSPACE}/inverno-core/doc/reference-guide.md, resources = img, heading-offset=1 )
$include( source=${INVERNO_WORKSPACE}/inverno-mods/doc/reference-guide.md, heading-offset=1 )
$include( source=${INVERNO_WORKSPACE}/inverno-mods/inverno-base/README.md, heading-offset=2 )
$include( source=${INVERNO_WORKSPACE}/inverno-mods/inverno-boot/README.md, heading-offset=2 )
$include( source=${INVERNO_WORKSPACE}/inverno-mods/inverno-configuration/README.md, heading-offset=2 )
$include( source=${INVERNO_WORKSPACE}/inverno-mods/inverno-discovery/README.md, heading-offset=2 )
$include( source=${INVERNO_WORKSPACE}/inverno-mods/inverno-discovery-http/README.md, heading-offset=2 )
$include( source=${INVERNO_WORKSPACE}/inverno-mods/inverno-discovery-http-meta/README.md, heading-offset=2 )
$include( source=${INVERNO_WORKSPACE}/inverno-mods/inverno-discovery-http-k8s/README.md, heading-offset=2 )
$include( source=${INVERNO_WORKSPACE}/inverno-mods/inverno-http-base/README.md, heading-offset=2 )
$include( source=${INVERNO_WORKSPACE}/inverno-mods/inverno-http-client/README.md, heading-offset=2 )
$include( source=${INVERNO_WORKSPACE}/inverno-mods/inverno-http-server/README.md, heading-offset=2 )
$include( source=${INVERNO_WORKSPACE}/inverno-mods/inverno-web-client/README.md, heading-offset=2 )
$include( source=${INVERNO_WORKSPACE}/inverno-mods/inverno-web-server/README.md, resources = doc/img, heading-offset=2 )
$include( source=${INVERNO_WORKSPACE}/inverno-mods/inverno-grpc-base/README.md, heading-offset=2 )
$include( source=${INVERNO_WORKSPACE}/inverno-mods/inverno-grpc-client/README.md, heading-offset=2 )
$include( source=${INVERNO_WORKSPACE}/inverno-mods/inverno-grpc-server/README.md, heading-offset=2 )
$include( source=${INVERNO_WORKSPACE}/inverno-mods/inverno-irt/README.md, resources = doc/img, heading-offset=2 )
$include( source=${INVERNO_WORKSPACE}/inverno-mods/inverno-sql/README.md, heading-offset=2 )
$include( source=${INVERNO_WORKSPACE}/inverno-mods/inverno-sql-vertx/README.md, heading-offset=3 )
$include( source=${INVERNO_WORKSPACE}/inverno-mods/inverno-redis/README.md, heading-offset=2 )
$include( source=${INVERNO_WORKSPACE}/inverno-mods/inverno-redis-lettuce/README.md, heading-offset=3 )
$include( source=${INVERNO_WORKSPACE}/inverno-mods/inverno-ldap/README.md, heading-offset=2 )
$include( source=${INVERNO_WORKSPACE}/inverno-mods/inverno-security/README.md, heading-offset=2 )
$include( source=${INVERNO_WORKSPACE}/inverno-mods/inverno-security-http/README.md, resources = doc/img, heading-offset=2 )
$include( source=${INVERNO_WORKSPACE}/inverno-mods/inverno-security-ldap/README.md, heading-offset=2 )
$include( source=${INVERNO_WORKSPACE}/inverno-mods/inverno-security-jose/README.md, heading-offset=2 )
$include( source=${INVERNO_WORKSPACE}/inverno-tools/inverno-build-tools/README.md, heading-offset=1 )
$include( source=${INVERNO_WORKSPACE}/inverno-tools/inverno-grpc-protoc-plugin/README.md, heading-offset=1 )
$include( source=${INVERNO_WORKSPACE}/inverno-tools/inverno-maven-plugin/README.md, heading-offset=1 )
$include( source=${INVERNO_WORKSPACE}/inverno-oss-parent/README.md, heading-offset=1 )
