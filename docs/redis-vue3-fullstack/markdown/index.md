$properties(base = ../../../, title = Full-Stack application Guide)

<div class="heading">
	<h1 class="heading-title">Inverno Framework Full-Stack application Guide</h1> 
	<p class="heading-subtitle">Author: <a href="mailto:jeremy.kuhn@inverno.io">Jeremy Kuhn</a></p> 
</div>

<div class="row align-items-stretch mt-5 mb-2">
	<div class="col-12 col-lg-6 mb-3">
		<div class="card shadow h-100">
			<div class="card-body p-lg-5">
				<h2 class="card-title">What you'll learn</h2>
				<p class="card-text">This guide demonstrates how the Inverno framework can be used to develop a full-stack application exposing REST services, accessing data in <a href="https://redis.io">Redis</a> data store and exposing a <a href="https://v3.vuejs.org/">Vue.js</a> front-end.</p>
				<p class="card-text">The application consists in a Task/Ticket Management System which demonstrates multiple Inverno's features such as: IoC/DI, configuration, Web Controllers, automatic OpenAPI specifications, WebJars and static resources, Redis client, <a href="https://www.docker.com/">Docker</a> packaging and deployment to the cloud.</p>
				<p class="card-text">We will guide you through the setup of a Maven project containing the Inverno module of the application, configuration setup, TLS setup, the creation of backend services to access Redis data store, the creation of Web controllers to expose these services as REST resources, the setup of Web routes to expose WebJars and static resources, running the application and finally packaging and deploying the application to <a href="https://www.docker.com/">Docker</a>.</p>
			</div>
		</div>
	</div>
	<div class="col-12 col-lg-6 mb-3">
		<div class="card shadow h-100">
			<div class="card-body p-lg-5">
				<h2 class="card-title">What you'll need</h2>
				<ul>
					<li>A <em>Java™ Development Kit</em> (<a href="https://openjdk.java.net/install/">OpenJDK</a>) at least version 16.</li>
					<li>Apache <a href="https://maven.apache.org/">Maven</a> at least version 3.6.</li>
					<li>An <em>Integrated Development Environment</em> (IDE) such as <a href="https://www.eclipse.org/">Eclipse</a> or <a href="https://www.jetbrains.com/idea/">IDEA</a> although any text editor will do.</li>
					<li>A <a href="https://www.docker.com/">Docker</a> installation.</li>
					<li>A basic understanding of <a href="https://redis.io">Redis</a> data store.</li>
					<li>A basic understanding of Inverno's core IoC/DI framework (see <a href="${base}docs/getting-started/html/index.html">Getting Started guide</a>).</li>
					<li>A basic understanding of reactive programming using <a href="https://projectreactor.io/">Project reactor</a> library.</li>
				</ul>
			</div>
		</div>
	</div>
</div>

$doc

## Step 0: Inverno Ticket application

The application you'll be creating is a simple Task or Ticket Management System, it allows grouping and organizing tickets into plans.

A **Ticket** represents a task which can be a feature to implement or an issue to fix. It has a status which can be one of: OPEN, STUDIED, IN_PROGRESS, DONE or REJECTED. One or more notes can be attached to a ticket to keep track of thoughts, progress or decisions.

A **Plan** organizes multiple related tickets with the aim of achieving a greater goal defined by the plan. Inside a plan, tickets can be arbitrarily sorted, they can then be organized in multiple ways depending on the plan (e.g. by priority, by importance, by complexity...). A given ticket can be linked to multiple plans.

Both **Plan** and **Ticket** have a title, a summary and a description.

Since a picture is worth a thousand words, the following wireframe shows what the application might look like in the end:

<img class="shadow" src="img/wireframe.svg" style="display: block; margin: 2em auto;" alt="Inverno Ticket wireframe"/>

The architecture is that of a typical Web application using [Redis](https://redis.io) as data store, defining a service layer with services to access the data store, a REST layer to expose those services to the front-end which consists in a single page application built with [Bootstrap](https://getbootstrap.com/) and [Vue.js](https://v3.vuejs.org/). 

All static resources, including front-end libraries, will be exposed by the application. In addition, an [OpenAPI](https://www.openapis.org/) specification of the REST API will be automatically generated and exposed in [SwaggerUI](https://swagger.io/tools/swagger-ui/).

The application will be eventually packaged and deployed to local [Docker](https://www.docker.com/) repository and run using [Docker Compose](https://docs.docker.com/compose/).

The full source code of the resulting application can be found in [GitHub](https://github.com/inverno-io/inverno-apps/tree/1.1.0/inverno-ticket).

## Step 1: Bootstrap the application project

The first thing to do is to create an Inverno module project which is a regular Maven Java project setup with Inverno distribution including an Inverno module descriptor and, since this is an application, an application entry point bootstrapping the Inverno module. 

You can start by creating a Maven Java project with groupId `io.inverno.guide` and artifactId `ticket` using your IDE, the Maven quickstart archetype or manually using the text editor of your choice. Whatever you choose, you should end up with a project folder with the following file structure as generated by the Maven quickstart archetype:

```text
├── pom.xml
└── src
    └── main
        └── java
            └── io
                └── inverno
                    └── guide
                        └── ticket
                            └── App.java
```

You can now set up the `pom.xml` build descriptor with the Inverno distribution by defining a `<parent/>` section pointing to `io.inverno.dist:inverno-parent:${VERSION_INVERNO_DIST}` parent pom:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>io.inverno.dist</groupId>
        <artifactId>inverno-parent</artifactId>
        <version>${VERSION_INVERNO_DIST}</version>
    </parent>
    <groupId>io.inverno.guide</groupId>
    <artifactId>ticket</artifactId>
    <version>1.0-SNAPSHOT</version>

</project>
```

Then you must declare a dependency to Inverno *boot* module which provides common services required by any Inverno application including core IoC/DI used to assemble application components at compile time and the unified configuration API used to configure applications as well as unified access to resources (file, Jar, classpath, module...), URI manipulation, global JSON Object mapper, data conversion, reactor, network service... 

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>io.inverno.dist</groupId>
        <artifactId>inverno-parent</artifactId>
        <version>${VERSION_INVERNO_DIST}</version>
    </parent>
    <groupId>io.inverno.guide</groupId>
    <artifactId>ticket</artifactId>
    <version>1.0-SNAPSHOT</version>

    <dependencies>
        <dependency>
            <groupId>io.inverno.mod</groupId>
            <artifactId>inverno-boot</artifactId>
        </dependency>
    </dependencies>
</project>
```

> Throughout this guide, you will add more dependencies in order to implement the service layer, the REST layer and the front-end layer.

You can now open up the project in your IDE and create a module descriptor `src/main/java/module-info.java`. In this module descriptor, you need to annotate the module statement with `@io.inverno.core.annotation.Module` to make it an Inverno module and declare the dependency to the `io.inverno.mod.boot` module:

```java
@io.inverno.core.annotation.Module
module io.inverno.guide.ticket {
    requires io.inverno.mod.boot;
}
```

You can now compile the project to generate the Inverno module class `io.inverno.guide.ticket.Ticket` in `target/generated-sources/annotations`:

```text
$ mvn compile
...
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  1.702 s
[INFO] Finished at: 2022-02-11T14:54:06+01:00
[INFO] ------------------------------------------------------------------------
```

You should now see the generated `Ticket` class in your IDE, this class will contain the IoC/DI logic used to bootstrap and assemble the application components.

You can now edit the application entry point `src/main/java/io/inverno/guide/ticket/App.java` and bootstrap the ticket module:

```java
package io.inverno.guide.ticket;

import io.inverno.core.v1.Application;

public class App {
    public static void main( String[] args ) {
        Application.run(new Ticket.Builder());
    }
}
```

The application can now be run with the following command:

```text
$ mvn invenro:run
...
[INFO] --- inverno-maven-plugin:${VERSION_INVERNO_TOOLS}:run (default-cli) @ ticket ---
[INFO] Running project: io.inverno.guide.ticket@1.0-SNAPSHOT...
ERROR StatusLogger Log4j2 could not find a logging implementation. Please add log4j-core to the classpath. Using SimpleLogger to log to the console...
INFO Application Inverno is starting...


     ╔════════════════════════════════════════════════════════════════════════════════════════════╗
     ║                      , ~~ ,                                                                ║
     ║                  , '   /\   ' ,                                                            ║
     ║                 , __   \/   __ ,      _                                                    ║
     ║                ,  \_\_\/\/_/_/  ,    | |  ___  _    _  ___   __  ___   ___                 ║
     ║                ,    _\_\/_/_    ,    | | / _ \\ \  / // _ \ / _|/ _ \ / _ \                ║
     ║                ,   __\_/\_\__   ,    | || | | |\ \/ /|  __/| | | | | | |_| |               ║
     ║                 , /_/ /\/\ \_\ ,     |_||_| |_| \__/  \___||_| |_| |_|\___/                ║
     ║                  ,     /\     ,                                                            ║
     ║                    ,   \/   ,                                 -- ${VERSION_INVERNO_CORE} --                  ║
     ║                      ' -- '                                                                ║
     ╠════════════════════════════════════════════════════════════════════════════════════════════╣
     ║ Java runtime        : OpenJDK Runtime Environment                                          ║
     ║ Java version        : 17+35-2724                                                           ║
     ║ Java home           : /home/jkuhn/Devel/jdk/jdk-17                                         ║
     ║                                                                                            ║
     ║ Application module  : io.inverno.guide.ticket                                              ║
     ║ Application version : 1.0-SNAPSHOT                                                         ║
     ║ Application class   : io.inverno.guide.ticket.App                                          ║
     ║                                                                                            ║
     ║ Modules             :                                                                      ║
     ║  * ...                                                                                     ║
     ║  * io.inverno.guide.ticket@1.0-SNAPSHOT                                                    ║
     ║  * ...                                                                                     ║
     ╚════════════════════════════════════════════════════════════════════════════════════════════╝


INFO Ticket Starting Module io.inverno.guide.ticket...
INFO Boot Starting Module io.inverno.mod.boot...
INFO Boot Module io.inverno.mod.boot started in 278ms
INFO Ticket Module io.inverno.guide.ticket started in 282ms
INFO Application Application io.inverno.guide.ticket started in 335ms
INFO Ticket Stopping Module io.inverno.guide.ticket...
INFO Boot Stopping Module io.inverno.mod.boot...
INFO Boot Module io.inverno.mod.boot stopped in 0ms
INFO Ticket Module io.inverno.guide.ticket stopped in 4ms
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  4.667 s
[INFO] Finished at: 2022-02-11T15:08:33+01:00
[INFO] ------------------------------------------------------------------------
```

## Step 2: Setup application configuration

Being able to inject configuration to your application in various ways depending on arbitrary factors will make it more flexible and more resilient to change. The Inverno framework provides a unified configuration module which facilitates application configuration from low-level configuration (e.g. host, port, log levels...) to higher level configuration (e.g. tenant specific, user preferences...).

The configuration module provides components that can be used in various situations to make your application code configurable. In order to operate the application on various platforms (e.g. bare metal, Docker, Kubernetes) and environment (e.g. development, test, production), it is important to make the application components fully configurable, this can be done easily by creating an application configuration interface and injecting a proper configuration source into the application module.

You can create the following `AppConfiguration` interface annotated with `@io.inverno.mod.configuration.Configuration` to define the application configuration:

```java
package io.inverno.guide.ticket;

import io.inverno.core.annotation.NestedBean;
import io.inverno.mod.boot.BootConfiguration;
import io.inverno.mod.configuration.Configuration;

@Configuration
public interface AppConfiguration {

    @NestedBean
    BootConfiguration boot();
}
```

In above configuration, the `BootConfiguration` has been declared as a nested bean in order to expose the boot module configuration.

You now need to inject a configuration source into the application module. A configuration source typically holds configuration data and exposes them to the application. In order to inject a configuration source into the application module, you need to create a socket bean by declaring a nested interface `AppConfigurationSource` in the application entry point as follows:

```java

package io.inverno.guide.ticket;

import io.inverno.core.annotation.Bean;
import io.inverno.core.v1.Application;
import io.inverno.mod.configuration.ConfigurationSource;

public class App {

    @Bean( name = "configurationSource" )
    public interface AppConfigurationSource extends Supplier<ConfigurationSource<?, ?, ?>> {}
    
    public static void main( String[] args ) {
        Application.run(new Ticket.Builder());
    }
}
```

You can now recompile the project to regenerate the Inverno module class and generate the application configuration loader `io.inverno.guide.ticket.AppConfigurationLoader` which loads the `appConfiguration` bean exposing configuration data to the module.

The Inverno configuration module provides multiple configuration source implementations that can be used in various contexts. The `BootstrapConfigurationSource` is particularly suited for bootstrapping an application, it scans the following local sources in that order to resolve configuration properties: 

- command line argument
- system properties
- system environment variables
- the `configuration.cprops` file in `./conf/` or `${inverno.config.path}/` directories if one exists (if the first one exists the second one is ignored)
- the `configuration.cprops` file in `${java.home}/conf/` directory if it exists 
- the `configuration.cprops` file in the application module if it exists

It is then possible to define a default configuration in a configuration file inside the module and override any properties from the command line, a system property or an external configuration file.

Let's create a `BootstrapConfigurationSource` and inject it into the application module:

```java
package io.inverno.guide.ticket;

import io.inverno.core.annotation.Bean;
import io.inverno.core.v1.Application;
import io.inverno.mod.configuration.ConfigurationSource;
import io.inverno.mod.configuration.source.BootstrapConfigurationSource;

import java.io.IOException;
import java.util.function.Supplier;

public class App {

    @Bean( name = "configurationSource" )
    public interface AppConfigurationSource extends Supplier<ConfigurationSource<?, ?, ?>> {}

    public static void main( String[] args ) throws IOException {
        Application.run(new Ticket.Builder().setConfigurationSource(new BootstrapConfigurationSource(App.class.getModule(), args)));
    }
}
```

Throughout this guide, you will provide default and specific configuration to the application components, so you can already create a `configuration.cprops` file under `src/main/resources`, this file will be packaged inside the module and is meant to contain generic application configuration.

```text
io.inverno.guide.ticket.appConfiguration {
    
}
```

the `.cprops` configuration file format is a specific file format which allows declaring namespaced and parameterized configuration properties as defined in the configuration module. In that particular case, the configuration properties namespace is `io.inverno.guide.ticket.appConfiguration` which corresponds to the name of the application module and the name of the application configuration bean.

> Please refer to the Inverno configuration module [configuration](https://inverno.io/docs/release/reference/html/index.html#configuration-1), to have a complete understanding on configuration sources, parameterized properties and the `.cprops` file format. 

## Step 3: Create the application Data model

As described earlier, the ticket application is dealing with the following entities:

- A **Plan** is characterized by a title, a summary and a description, and it can have one or more associated tickets.
- A **Ticket** is characterized by a type, a status, a title, a summary and a description, and it can be associated to one or more plans and have zero or more notes.
- A **Note** is characterized by a title and a content, and it is linked to exactly one ticket.

This can be modelled in the following diagram:

<img class="shadow" src="img/datamodel.svg" style="display: block; margin: 2em auto;" alt="Inverno Ticket data model"/>

Plans and tickets are uniquely identified by generated ids, ticket notes are stored as list associated to a ticket, they are then uniquely identified by a ticket id and an index. 

You can start by creating the `io.inverno.guide.ticket.internal.model.Plan` class as follows:

```java
package io.inverno.guide.ticket.internal.model;

import com.fasterxml.jackson.annotation.JsonIgnore;
import com.fasterxml.jackson.annotation.JsonInclude;
import reactor.core.publisher.Flux;

import java.time.ZonedDateTime;

@JsonInclude(JsonInclude.Include.NON_NULL)
public class Plan {

    private Long id;
    private String title;
    private String summary;
    private String description;
    private ZonedDateTime creationDateTime;
    @JsonIgnore
    private Flux<Ticket> tickets;

    // Constructors
    // Getters, Setters
}
```

The `tickets` one-to-many relationship has been declared as a `Flux<Ticket>`, Inverno is fully reactive, using a `Flux` here allows to lazily fetch the tickets associated to a plan which can be very convenient.

Some Jackson annotations are also specified to indicate how data should be deserialized and serialized from/to the data store. For instance, the `@JsonIgnore` annotation specifies that tickets, which will be stored in a dedicated entry should be ignored by Jackson.

You can move on to the creation of the `io.inverno.guide.ticket.internal.model.Ticket` class as follows:

```java
package io.inverno.guide.ticket.internal.model;

import com.fasterxml.jackson.annotation.JsonInclude;

import java.time.ZonedDateTime;

@JsonInclude(JsonInclude.Include.NON_NULL)
public class Ticket {

    public enum Type {
        FEATURE,
        ISSUE
    }

    public enum Status {
        OPEN,
        STUDIED,
        IN_PROGRESS,
        DONE,
        REJECTED
    }

    private Long id;
    private Type type;
    private Status status;
    private ZonedDateTime creationDateTime;
    private String title;
    private String summary;
    private String description;

    // Constructors
    // Getters, Setters
}
```

A ticket can be associated to multiple plans and have multiple notes, but a ticket exists on its own, these relationships are unidirectional and as a result there is no reference to plan or note in the `Ticket` class.

Finally, you can create the `io.inverno.guide.ticket.internal.model.Note` class as follows:

```java
package io.inverno.guide.ticket.internal.model;

import com.fasterxml.jackson.annotation.JsonInclude;

@JsonInclude(JsonInclude.Include.NON_NULL)
public class Note {

    private long ticketId;
    private Integer index;
    private String title;
    private String content;

    // Constructors
    // Getters, Setters
}
```

A note only exists in the context of a ticket and as a result the id of the ticket must be present in the `Note` class. 

Finally, in order for the Jackson object mapper to be able to serialize/deserialize the data model, you must add the following `exports` directive in the module descriptor to allow Jackson to access the data model.

```java
@io.inverno.core.annotation.Module
module io.inverno.guide.ticket {
    requires io.inverno.mod.boot;
    requires io.inverno.mod.redis.lettuce;

    exports io.inverno.guide.ticket.internal.model to com.fasterxml.jackson.databind;
}
```

## Step 4: Create the Service layer

You can now move on to the service layer which provides services to create, read, update and delete above entities in Redis data store. 

Inverno provides a [Redis client API](https://inverno.io/docs/release/reference/html/index.html#redis-client) which can be used to access a Redis data store in a reactive way consistent with the rest of the framework. In order to use the Redis client you need to declare a dependency to the Redis client implementation module in the Maven project descriptor and in the Java module descriptor.

Inverno currently provides an implementation based on [Lettuce](https://lettuce.io/).

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>io.inverno.dist</groupId>
        <artifactId>inverno-parent</artifactId>
        <version>${VERSION_INVERNO_DIST}</version>
    </parent>
    <groupId>io.inverno.guide</groupId>
    <artifactId>ticket</artifactId>
    <version>1.0-SNAPSHOT</version>

    <dependencies>
        <dependency>
            <groupId>io.inverno.mod</groupId>
            <artifactId>inverno-boot</artifactId>
        </dependency>
        <dependency>
            <groupId>io.inverno.mod</groupId>
            <artifactId>inverno-redis-lettuce</artifactId>
        </dependency>
    </dependencies>
</project>
```

You must also declare the dependency in the `module-info.java` descriptor of the project.

```java
@io.inverno.core.annotation.Module
module io.inverno.guide.ticket {
    requires io.inverno.mod.boot;
    requires io.inverno.mod.redis.lettuce;
}
```

`io.inverno.mod.redis.lettuce` is an Inverno module exposing a `RedisTransactionalClient<String, String>` bean and a `LettuceRedisClientConfiguration` configuration bean, it can then be injected in any bean defined in the application module.

In order to be able to configure the Redis client, the `LettuceRedisClientConfiguration` must be exposed in the `AppConfiguration`:

```java
package io.inverno.guide.ticket;

import io.inverno.core.annotation.NestedBean;
import io.inverno.mod.boot.BootConfiguration;
import io.inverno.mod.configuration.Configuration;
import io.inverno.mod.redis.lettuce.LettuceRedisClientConfiguration;

@Configuration
public interface AppConfiguration {

    @NestedBean
    BootConfiguration boot();

    @NestedBean
    LettuceRedisClientConfiguration redis();
}
```

Since the `LettuceRedisClientConfiguration` is declared as a nested bean in the `AppConfiguration`, it will be automatically injected in the Lettuce Redis client module and used to configure the Redis client.

You can now create three services to manage plans, tickets and notes. The following naming strategy will be used for keys in the Redis data store:

```
APP:<APPLICATION>:<DATA_TYPE>:<ID>[:<OPT>]*
```

- `<APPLICATION>` uniquely identifies the application, in your case: `Ticket`
- `<DATA_TYPE>` designates the type of data: `Plan`, `Ticket` or `Note`
- `<ID>` uniquely identifies the data, it can be a sequence for a plan or a ticket (e.g. `APP:Ticket:Ticket:2`) but it can also be a constant to identify single entries such as sequences (eg. `APP:Ticket:Ticket:SEQ`)
- `<OPT>` could be any extra metadata used to designate any data related to a parent entry such as the list of tickets associated to a plan (e.g. `APP:Ticket:Plan:2:Tickets`)

Since the `APP:Ticket` prefix is common to the whole application, you can declare it in the application entry point:

```java
package io.inverno.guide.ticket;

public class App {

    public static final String REDIS_KEY = "APP:Ticket";
    
    ...
}
```

The `io.inverno.guide.ticket.internal.service.TicketService` class manages tickets in the Redis data store. It must be annotated with `@io.inverno.core.annotation.Bean` to take part in IoC/DI when the application module is started. It requires a `RedisTransactionalClient<String, String>` instance to interacts with Redis and an `ObjectMapper` instance to serialize/deserialize `Ticket`. These are required dependencies that must be injected in the constructor.

> The `RedisTransactionalClient<String, String>` dependency is provided by the `io.inverno.mod.redis.lettuce` module and the `ObjectMapper` dependency is provided by the `io.inverno.mod.boot` module.

The `TicketService` exposes basic operations to create, read, update and delete tickets in the data store. 

```java
package io.inverno.guide.ticket.internal.service;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;
import io.inverno.core.annotation.Bean;
import io.inverno.guide.ticket.App;
import io.inverno.guide.ticket.internal.exception.TicketException;
import io.inverno.guide.ticket.internal.model.Ticket;
import io.inverno.mod.redis.RedisTransactionalClient;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

import java.io.IOException;
import java.io.UncheckedIOException;
import java.time.ZoneOffset;
import java.time.ZonedDateTime;
import java.util.List;

@Bean
public class TicketService {

    private final RedisTransactionalClient<String, String> redisClient;
    private final ObjectMapper mapper;

    public TicketService(RedisTransactionalClient<String, String> redisClient, ObjectMapper mapper) {
        this.redisClient = redisClient;
        this.mapper = mapper;
    }

    public Mono<Ticket> saveTicket(Ticket ticket) {...}

    public Mono<Ticket> updateTicketStatus(long ticketId, Ticket.Status status) {...}

    public Flux<Ticket> listTickets() {...}

    public Flux<Ticket> listTickets(List<Ticket.Status> statuses) {...}

    public Mono<Ticket> getTicket(long ticketId) {...}

    public Flux<Ticket> getTickets(List<Long> ticketIds) {...}

    public Mono<Ticket> removeTicket(long ticketId) {...}
}
```

A ticket is stored as a Redis `string`, it is uniquely identified by an id generated by a dedicated sequence `APP:Ticket:Ticket:SEQ` using `INCR` Redis command. A ticket is then stored at key `APP:Ticket:Ticket:2` where `2` is the id of the ticket. Besides, it should be possible to list and filter tickets by status, as a result several Redis `set`, one per status (e.g. `APP:Ticket:Ticket:OPEN`), are used to label tickets and easily filter tickets by status using `SUNION` Redis command.

Let's see in details how this works in the `saveTicket()` method which creates or updates a ticket and returns the resulting ticket:

```java
package io.inverno.guide.ticket.internal.service;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;
import io.inverno.core.annotation.Bean;
import io.inverno.guide.ticket.App;
import io.inverno.guide.ticket.internal.exception.TicketException;
import io.inverno.guide.ticket.internal.model.Ticket;
import io.inverno.mod.redis.RedisTransactionalClient;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

import java.io.IOException;
import java.io.UncheckedIOException;
import java.time.ZoneOffset;
import java.time.ZonedDateTime;
import java.util.List;

@Bean
public class TicketService {

    public static final String REDIS_KEY_TICKET = App.REDIS_KEY + ":Ticket:%d";
    public static final String REDIS_KEY_TICKET_SEQ = App.REDIS_KEY + ":Ticket:SEQ";
    
    public static final String REDIS_KEY_TICKET_STATUS = App.REDIS_KEY + ":Ticket:%s";
    public static final String REDIS_KEY_TICKET_OPEN = String.format(REDIS_KEY_TICKET_STATUS, Ticket.Status.OPEN);
    public static final String REDIS_KEY_TICKET_STUDIED = String.format(REDIS_KEY_TICKET_STATUS, Ticket.Status.STUDIED);
    public static final String REDIS_KEY_TICKET_IN_PROGRESS = String.format(REDIS_KEY_TICKET_STATUS, Ticket.Status.IN_PROGRESS);
    public static final String REDIS_KEY_TICKET_DONE = String.format(REDIS_KEY_TICKET_STATUS, Ticket.Status.DONE);
    public static final String REDIS_KEY_TICKET_REJECTED = String.format(REDIS_KEY_TICKET_STATUS, Ticket.Status.REJECTED);
    
    ...

    public Mono<Ticket> saveTicket(Ticket ticket) {
        if(ticket.getId() != null) {
            // Try to update
            return Mono.from(this.redisClient.connection(operations -> {
                try {
                    return operations
                        .setGet()
                        .xx()
                        .build(String.format(REDIS_KEY_TICKET, ticket.getId()), this.mapper.writeValueAsString(ticket))
                        .flatMap(result -> {
                            try {
                                Ticket oldTicket = this.mapper.readValue(result, Ticket.class);
                                if(!oldTicket.getStatus().equals(ticket.getStatus())) {
                                    return operations.smove(String.format(REDIS_KEY_TICKET_STATUS, oldTicket.getStatus()), String.format(REDIS_KEY_TICKET_STATUS, ticket.getStatus()), Long.toString(ticket.getId())).thenReturn(ticket);
                                }
                                else {
                                    return Mono.just(ticket);
                                }
                            }
                            catch (JsonProcessingException ex) {
                                throw new UncheckedIOException(ex);
                            }
                        });
                }
                catch (JsonProcessingException ex) {
                    throw new UncheckedIOException(ex);
                }
            }));
        }
        else {
            return this.redisClient
                .incr(REDIS_KEY_TICKET_SEQ)
                .flatMap(ticketId -> {
                    ticket.setCreationDateTime(ZonedDateTime.now(ZoneOffset.UTC));
                    ticket.setStatus(Ticket.Status.OPEN);
                    ticket.setId(ticketId);
                    return this.redisClient.multi(operations -> {
                        try {
                            return Flux.just(
                                operations
                                    .set()
                                    .nx()
                                    .build(String.format(REDIS_KEY_TICKET, ticketId), this.mapper.writeValueAsString(ticket)),
                                operations
                                    .sadd(REDIS_KEY_TICKET_OPEN, Long.toString(ticketId))
                            );
                        }
                        catch (JsonProcessingException ex) {
                            throw new UncheckedIOException(ex);
                        }
                    });
                })
                .map(transactionResult -> {
                    if(transactionResult.wasDiscarded()) {
                        throw new TicketException("Error while creating ticket: transaction was discarded");
                    }
                    return ticket;
                });
        }
    }
}
```

The `RedisClient` provides method `connection()` which allows running multiple Redis commands on a single connection taken from a connection pool. The `RedisOperations` instance thus obtained is passed to the argument function to run Redis commands, the connection is automatically pushed back to the pool once the `Publisher` returned by that function terminates. A command can also be run directly on the client instance. Above method demonstrates both approaches.

> Invoking multiple commands on the client instance might be less performant since one connection is then requested to the pool for each command. But as you can see this can be convenient when considering a single command or when the client implementation uses a single connection.

The `RedisTransactionalClient` provides method `multi()` which allows running multiple Redis commands within a Redis *transaction*. As for the `connection()` method, a connection is taken from a connection pool and a transaction started. The `RedisOperations` instance thus obtained is passed to the argument function and used to create multiple Redis commands emitted by the `Publisher` returned by that function. The transaction is executed or discarded and the connection returned to the pool once all command publishers terminates, a `TransactionResult` is then returned. 

A ticket is updated when a ticket id is present in the ticket argument by updating the JSON representation at the key corresponding to the ticket id and by moving the ticket id from the old status set to the new status set if the ticket status was actually updated. This is done using `SET` and `SMOVE` Redis commands. Note that the `xx` option is used when updating the ticket, as a result a non-existing ticket results in an empty Mono.

A new ticket is created when there's no ticket id in the ticket argument. A ticket sequence is first obtained by incrementing `APP:Ticket:Ticket:SEQ` and then a JSON representation of the ticket is stored at the corresponding key. The ticket id is also added to the `APP:Ticket:Ticket:OPEN` set since a new ticket is always in status `OPEN`. These two commands are run within a transaction in order to make sure a ticket is always created and added to the `OPEN` ticket status set.

The Redis client API is quite self-explanatory, but we can differentiate between simple and complex commands: a simple command is run by subscribing to a `Publisher` directly returned by a `RedisOperations` method whereas a complex command is run by subscribing to a `Publisher` obtained from a builder returned by a `RedisOperations` method which allows defining more complex arguments. In above code, the `SADD` command is considered a simple command and the `SET` command a complex command.

The rest of the implementation is done in a similar way, the complete code can be found in [GitHub](https://github.com/inverno-io/inverno-apps/blob/1.1.0/inverno-ticket/src/main/java/io/inverno/app/ticket/internal/service/TicketService.java).

The `io.inverno.guide.ticket.internal.service.PlanService` class manages plans in the Redis data store. It must be annotated with `io.inverno.core.annotation.Bean` to take part in IoC/DI when the application module is started. As for the `TicketService` bean, it requires a `RedisTransactionalClient<String, String>` instance and an `ObjectMapper` instance but also a `TicketService` instance to retrieve tickets associated to a plan. These are required dependencies that must be injected in the constructor.

The `PlanService` exposes basic operations to create, read, update and delete plans in the data store but also to associate tickets to plan.

```java
package io.inverno.guide.ticket.internal.service;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;
import io.inverno.core.annotation.Bean;
import io.inverno.guide.ticket.App;
import io.inverno.guide.ticket.internal.exception.PlanAlreadyExistsException;
import io.inverno.guide.ticket.internal.exception.TicketException;
import io.inverno.guide.ticket.internal.exception.TicketNotFoundInPlanException;
import io.inverno.guide.ticket.internal.model.Plan;
import io.inverno.guide.ticket.internal.model.Ticket;
import io.inverno.mod.redis.RedisTransactionalClient;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

import java.io.IOException;
import java.io.UncheckedIOException;
import java.time.ZoneOffset;
import java.time.ZonedDateTime;
import java.util.List;

@Bean
public class PlanService {

    private final RedisTransactionalClient<String, String> redisClient;
    private final ObjectMapper mapper;
    private final TicketService ticketService;

    public PlanService(RedisTransactionalClient<String, String> redisClient, ObjectMapper mapper, TicketService ticketService) {
        this.redisClient = redisClient;
        this.mapper = mapper;
        this.ticketService = ticketService;
    }

    public Mono<Plan> savePlan(Plan plan) {...}

    public Mono<Void> addTicket(long planId, long ticketId) {...}

    public Mono<Void> insertTicketBefore(long planId, long ticketId, long referenceTicketId) {...}

    public Mono<Long> removeTicket(long planId, long ticketId) {...}

    public Flux<Plan> listPlans() {...}

    public Mono<Plan> getPlan(long planId) {...}

    public Mono<Plan> getPlan(long planId, List<Ticket.Status> statuses) {...}

    public Mono<Plan> removePlan(long planId) {...}
}
```

A plan is stored as a Redis `string` and uniquely identified by an id generated by a dedicated sequence `APP:Ticket:Plan:SEQ` using `INCR` Redis command. A plan is then stored at key `APP:Ticket:Plan:2` where `2` is the id of the plan. Besides, a Redis `list` is used to record and order the tickets associated to a plan. This list is directly related to the plan and its key then derives from the plan key: `APP:Ticket:Plan:2:Tickets`.

Let's take a closer look at the `getPlan()` methods to see how the tickets associated to a plan are lazily loaded.

```java
package io.inverno.guide.ticket.internal.service;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;
import io.inverno.core.annotation.Bean;
import io.inverno.guide.ticket.App;
import io.inverno.guide.ticket.internal.exception.PlanAlreadyExistsException;
import io.inverno.guide.ticket.internal.exception.TicketException;
import io.inverno.guide.ticket.internal.exception.TicketNotFoundInPlanException;
import io.inverno.guide.ticket.internal.model.Plan;
import io.inverno.guide.ticket.internal.model.Ticket;
import io.inverno.mod.redis.RedisTransactionalClient;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

import java.io.IOException;
import java.io.UncheckedIOException;
import java.time.ZoneOffset;
import java.time.ZonedDateTime;
import java.util.List;

@Bean
public class PlanService {

    public static final String REDIS_KEY_PLAN = App.REDIS_KEY + ":Plan:%d";
    public static final String REDIS_KEY_PLAN_SEQ = App.REDIS_KEY + ":Plan:SEQ";

    public static final String REDIS_KEY_PLAN_TICKETS = App.REDIS_KEY + ":Plan:%d:Tickets";

    private static final String REDIS_KEY_PLAN_PATTERN = App.REDIS_KEY + ":Plan:*";
    private static final String REDIS_KEY_PLAN_REGEX = App.REDIS_KEY + ":Plan:[0-9]*";

    private final RedisTransactionalClient<String, String> redisClient;
    private final ObjectMapper mapper;
    private final TicketService ticketService;

    ...

    public Mono<Plan> getPlan(long planId, List<Ticket.Status> statuses) {
        return this.redisClient.get(String.format(REDIS_KEY_PLAN, planId))
                .map(result -> {
                    try {
                        Plan plan = this.mapper.readValue(result, Plan.class);
                        plan.setTickets(this.getPlanTickets(planId, statuses));
                        return plan;
                    } catch (JsonProcessingException ex) {
                        throw new UncheckedIOException(ex);
                    }
                });
    }

    private Flux<Ticket> getPlanTickets(long planId, List<Ticket.Status> statuses) {
        if (statuses == null || statuses.isEmpty()) {
            return Flux.empty();
        }
        return Flux.from(this.redisClient.connection(operations -> operations
                .sunion(keys -> statuses.forEach(status -> keys.key(String.format(TicketService.REDIS_KEY_TICKET_STATUS, status))))
                .collectList()
                .filter(ticketIds -> !ticketIds.isEmpty())
                .flatMapMany(ticketIds -> operations.lrange(String.format(REDIS_KEY_PLAN_TICKETS, planId), 0, -1)
                        .filter(id -> ticketIds.contains(id))
                        .map(id -> Long.parseLong(id))
                        .collectList()
                        .flatMapMany(this.ticketService::getTickets)
                )
        ));
    }
}
```

The `getPlan()` method allows to retrieve a plan with its associated tickets filtered by status and lazily loaded, it uses the `TicketService` to obtain the ticket `Publisher`.

Retrieving a plan is pretty straightforward using Redis `GET` command, the filtered tickets `Publisher` is then returned by the `getPlanTickets()` method which first gets the complete list of ticket ids that match the requested statuses, then gets the list of ticket ids associated to the plan and eventually returned the intersection.

Using a `Flux<Ticket>` allows to lazily load plan's ticket when required. This is one advantage of being reactive since nothing happens until the publisher is subscribed.

The rest of the `PlanService` implementation is done in a similar way as for the `TicketService`, the complete code can be found in [GitHub](https://github.com/inverno-io/inverno-apps/blob/1.1.0/inverno-ticket/src/main/java/io/inverno/app/ticket/internal/service/PlanService.java).

The `io.inverno.guide.ticket.internal.service.NoteService` class manages ticket notes in the Redis data store. It must be annotated with `io.inverno.core.annotation.Bean` to take part in IoC/DI when the application module is started. It requires a `RedisTransactionalClient<String, String>` instance and an `ObjectMapper` instance. These are required dependencies that must be injected in the constructor.

The `NoteService` exposes basic operations to create, read, update and delete ticket notes in the data store.

```java
package io.inverno.guide.ticket.internal.service;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;
import io.inverno.guide.ticket.internal.model.Note;
import io.inverno.mod.redis.RedisTransactionalClient;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

import java.io.UncheckedIOException;

public class NoteService {

    public static final String REDIS_KEY_TICKET_NOTES = TicketService.REDIS_KEY_TICKET + ":Notes";

    private final RedisTransactionalClient<String, String> redisClient;
    private final ObjectMapper mapper;

    public NoteService(RedisTransactionalClient<String, String> redisClient, ObjectMapper mapper) {
        this.redisClient = redisClient;
        this.mapper = mapper;
    }

    public Mono<Note> saveTicketNote(Note note) {...}

    public Flux<Note> listTicketNotes(long ticketId) {...}

    public Mono<Note> getTicketNote(long ticketId, int noteIndex) {...}

    public Mono<Note> removeTicketNote(long ticketId, int noteIndex) {...}
}
```

Notes associated to a ticket are stored in a Redis `list` whose key derives from the ticket id: `APP:Ticket:Ticket:2:Notes`.

The `NoteService` implementation is a basic CRUD implementation which is similar to what we've seen so far in `TicketService` and `PlanService`, the complete code can be found in [GitHub](https://github.com/inverno-io/inverno-apps/blob/1.1.0/inverno-ticket/src/main/java/io/inverno/app/ticket/internal/service/NoteService.java).

Inverno fully embraces the [Java Platform Module System](https://en.wikipedia.org/wiki/Java_Platform_Module_System) to create modular and secure applications. Unfortunately, not all Java libraries have been properly migrated to Java modules. This is especially the case for [Lettuce](https://lettuce.io/) and [Project Reactor](https://projectreactor.io/). This might result in self-explanatory runtime errors such as: `java.lang.reflect.InaccessibleObjectException: ... module reactor.core does not "opens reactor.core.publisher" to module lettuce.core`. Until external libraries are properly modularized, such issues can be fixed by specifying `--add-opens` or `--add-exports` arguments to the JVM.

Since you'll use the Inverno Maven plugin to run and package the application, you can fix this issue by configuring the plugin in the build descriptor as follows:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>io.inverno.dist</groupId>
        <artifactId>inverno-parent</artifactId>
        <version>${VERSION_INVERNO_DIST}</version>
    </parent>
    <groupId>io.inverno.guide</groupId>
    <artifactId>ticket</artifactId>
    <version>1.0-SNAPSHOT</version>

    ...

    <build>
        <pluginManagement>
            <plugins>
                <plugin>
                    <groupId>io.inverno.tool</groupId>
                    <artifactId>inverno-maven-plugin</artifactId>
                    <configuration>
                        <vmOptions>--add-opens reactor.core/reactor.core.publisher=lettuce.core -Dorg.apache.logging.log4j.simplelog.level=INFO -Dorg.apache.logging.log4j.level=INFO</vmOptions>
                    </configuration>
                </plugin>
            </plugins>
        </pluginManagement>
    </build>
</project>
```

> You also need to specify Log4j properties to configure the default Log4j Logger, logging will be set up later in this documentation and these won't be necessary anymore. 

## Step 5: Create the REST layer

The REST layer exposes previous service layer to front-ends or external systems as REST services. These endpoints should expose [Data Tranfer Objects](https://en.wikipedia.org/wiki/Data_transfer_object) rather than the Domain data model to be able to evolve the domain model without impacting front-ends or external systems and to optimize communication in a lesser extent considering that the model is pretty simple here. Both DTOs and REST endpoints should be versioned, again to be able to make the application evolve independently.

In practice, the ticket application shall expose two REST endpoints for plans and tickets versioned in the URI: `/api/v1/plan` and `/api/v1/ticket`.

Let's start by defining DTOs corresponding to `Plan`, `Ticket` and `Note`. These DTOs must be created within a versioned package `io.inverno.guide.ticket.internal.rest.v1.dto`.

The `PlanDto` class contains the same fields as the `Plan` class but since it is a DTO, whose purpose is serialization/deserialization, it doesn't contain any behaviour and as a result the `Flux<Ticket>` which allows to lazily load plan's tickets is replaced by a `List<TicketDto>`.

```java
package io.inverno.guide.ticket.internal.rest.v1.dto;

import java.time.ZonedDateTime;
import java.util.List;

public class PlanDto {

    private Long id;
    private String title;
    private String summary;
    private String description;
    private ZonedDateTime creationDateTime;
    private List<TicketDto> tickets;
    
    // Constructors
    // Getters, Setters
}
```

The `TicketDto` class is more basic and contains the same fields as the `Ticket` class.

```java
package io.inverno.guide.ticket.internal.rest.v1.dto;

import io.inverno.guide.ticket.internal.model.Ticket;

import java.time.ZonedDateTime;

public class TicketDto {

    private Long id;
    private Ticket.Type type;
    private Ticket.Status status;
    private ZonedDateTime creationDateTime;
    private String title;
    private String summary;
    private String description;
    
    // Constructors
    // Getters, Setters
}
```

The `NoteDto` class is also similar to the `Note` class.

```java
package io.inverno.guide.ticket.internal.rest.v1.dto;

public class NoteDto {

    private long ticketId;
    private Integer Index;
    private String title;
    private String content;

    // Constructors
    // Getters, Setters
}
```

DTOs must be converted to Domain objects and vice versa, so you'll need to define mappers for each of them. Many mapping libraries exist that can automate this task but let's keep things simple and define a simple reactive `DtoMapper` interface:

```java
package io.inverno.guide.ticket.internal.rest;

import reactor.core.publisher.Mono;

public interface DtoMapper<DTO, DOMAIN> {

    Mono<DTO> toDto(DOMAIN domain);

    Mono<DOMAIN> toDomain(DTO dto);
}
```

You can then implement `PlanDtoMapper`, `TicketDtoMapper` and `NoteDtoMapper`. These class should be declared as beans annotated with `@io.inverno.core.annotation.Bean` in order to be easily injected in REST controllers.

`TicketDtoMapper` and `NoteDtoMapper` are pretty simple to implement as this is basically a one-to-one mapping.

```java
package io.inverno.guide.ticket.internal.rest.v1.mapper;

import io.inverno.guide.ticket.internal.model.Ticket;
import io.inverno.guide.ticket.internal.rest.DtoMapper;
import io.inverno.guide.ticket.internal.rest.v1.dto.TicketDto;
import reactor.core.publisher.Mono;

@Bean( visibility = Bean.Visibility.PRIVATE )
public class TicketDtoMapper implements DtoMapper<TicketDto, Ticket> {

    @Override
    public Mono<TicketDto> toDto(Ticket domain) {
        return Mono.fromSupplier(() -> {
            TicketDto dto = new TicketDto();

            dto.setId(domain.getId());
            dto.setType(domain.getType());
            dto.setStatus(domain.getStatus());
            dto.setTitle(domain.getTitle());
            dto.setSummary(domain.getSummary());
            dto.setDescription(domain.getDescription());
            dto.setCreationDateTime(domain.getCreationDateTime());

            return dto;
        });
    }

    @Override
    public Mono<Ticket> toDomain(TicketDto dto) {
        return Mono.fromSupplier(() -> {
            Ticket domain = new Ticket();

            domain.setId(dto.getId());
            domain.setType(dto.getType());
            domain.setStatus(dto.getStatus());
            domain.setTitle(dto.getTitle());
            domain.setSummary(dto.getSummary());
            domain.setDescription(dto.getDescription());
            domain.setCreationDateTime(dto.getCreationDateTime());

            return domain;
        });
    }
}
```

The code of the `NoteDtoMapper` implementation can be found in [GitHub](https://github.com/inverno-io/inverno-apps/blob/1.1.0/inverno-ticket/src/main/java/io/inverno/app/ticket/internal/rest/v1/mapper/NoteDtoMapper.java).

The `PlanDtoMapper` is a bit more complex since `Flux<Ticket>` must be mapped to `List<TicketDto>`, implementation then requires some logic and a `DtoMapper<TicketDto, Ticket>` which can be easily injected in the constructor since both `PlanDtoMapper` and `TicketDtoMapper` are declared as beans in the same module.

```java
package io.inverno.guide.ticket.internal.rest.v1.mapper;

import io.inverno.core.annotation.Bean;
import io.inverno.guide.ticket.internal.model.Plan;
import io.inverno.guide.ticket.internal.model.Ticket;
import io.inverno.guide.ticket.internal.rest.DtoMapper;
import io.inverno.guide.ticket.internal.rest.v1.dto.PlanDto;
import io.inverno.guide.ticket.internal.rest.v1.dto.TicketDto;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

import java.util.Optional;

@Bean( visibility = Bean.Visibility.PRIVATE )
public class PlanDtoMapper implements DtoMapper<PlanDto, Plan> {

    private final DtoMapper<TicketDto, Ticket> ticketDtoMapper;

    public PlanDtoMapper(DtoMapper<TicketDto, Ticket> ticketDtoMapper) {
        this.ticketDtoMapper = ticketDtoMapper;
    }

    @Override
    public Mono<PlanDto> toDto(Plan domain) {
        return Optional.ofNullable(domain.getTickets()).orElse(Flux.empty())
                .flatMap(this.ticketDtoMapper::toDto)
                .collectList()
                .map(tickets -> {
                    PlanDto dto = new PlanDto();

                    dto.setId(domain.getId());
                    dto.setTitle(domain.getTitle());
                    dto.setSummary(domain.getSummary());
                    dto.setDescription(domain.getDescription());
                    dto.setCreationDateTime(domain.getCreationDateTime());
                    dto.setTickets(tickets);

                    return dto;
                });
    }

    @Override
    public Mono<Plan> toDomain(PlanDto dto) {
        return Mono.fromSupplier(() -> {
            Plan plan = new Plan();

            plan.setId(dto.getId());
            plan.setTitle(dto.getTitle());
            plan.setDescription(dto.getDescription());
            plan.setSummary(dto.getSummary());
            plan.setCreationDateTime(dto.getCreationDateTime());
            if(dto.getTickets() != null) {
                plan.setTickets(Flux.fromIterable(dto.getTickets()).flatMap(this.ticketDtoMapper::toDomain));
            }

            return plan;
        });
    }
}
```

You can now move to the creation of the two REST endpoints. Inverno provides a Web module that facilitates the creation of REST endpoints, you need then to declare a dependency to the Web module in the Maven project descriptor and in the Java module descriptor.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>io.inverno.dist</groupId>
        <artifactId>inverno-parent</artifactId>
        <version>${VERSION_INVERNO_DIST}</version>
    </parent>
    <groupId>io.inverno.guide</groupId>
    <artifactId>ticket</artifactId>
    <version>1.0-SNAPSHOT</version>

    <dependencies>
        <dependency>
            <groupId>io.inverno.mod</groupId>
            <artifactId>inverno-boot</artifactId>
        </dependency>
        <dependency>
            <groupId>io.inverno.mod</groupId>
            <artifactId>inverno-redis-lettuce</artifactId>
        </dependency>
        <dependency>
            <groupId>io.inverno.mod</groupId>
            <artifactId>inverno-web</artifactId>
        </dependency>
    </dependencies>
    
    ...
</project>
```

You must also declare the dependency in the `module-info.java` descriptor of the project.

```java
@io.inverno.core.annotation.Module
module io.inverno.guide.ticket {
    requires io.inverno.mod.boot;
    requires io.inverno.mod.redis.lettuce;
    requires io.inverno.mod.web;
}
```

`io.inverno.mod.web` is an Inverno module which embeds the HTTP server and allows defining Web routes used to route HTTP requests to the right handler. It can be configured by injecting a `WebConfiguration` which should then be exposed in the `AppConfiguration` as follows:

```java
package io.inverno.guide.ticket;

import io.inverno.core.annotation.NestedBean;
import io.inverno.mod.boot.BootConfiguration;
import io.inverno.mod.configuration.Configuration;
import io.inverno.mod.redis.lettuce.LettuceRedisClientConfiguration;
import io.inverno.mod.web.WebConfiguration;

@Configuration
public interface AppConfiguration {

    @NestedBean
    BootConfiguration boot();

    @NestedBean
    LettuceRedisClientConfiguration redis();

    @NestedBean
    WebConfiguration web();
}
```

Since the `WebConfiguration` is declared as a nested bean in the `AppConfiguration`, it will be automatically injected in the Web module and used to configure the HTTP server among other things.

The Web module provides several ways to create REST endpoint, it can be done by defining Web routes programmatically or by defining Web controllers later processed by the Inverno Web compiler at build time to generate Web router configurers injected in the Web module to configure the corresponding Web routes. As well as being simpler, using Web controllers allows generating [OpenAPI](https://www.openapis.org/) specifications automatically based on JavaDoc.

Let's start by creating the `PlanWebController` which exposes the `PlanService` in a REST interface. It must be annotated with both `@io.inverno.core.annotation.Bean` and `@io.inverno.mod.web.annotation.WebController` to make it a Web controller, it also requires a `PlanService` instance and a `DtoMapper<PlanDto, Plan>` instance which must be declared in the constructor as required dependencies.

```java
package io.inverno.guide.ticket.internal.rest.v1;

import io.inverno.core.annotation.Bean;
import io.inverno.guide.ticket.internal.model.Plan;
import io.inverno.guide.ticket.internal.model.Ticket;
import io.inverno.guide.ticket.internal.rest.DtoMapper;
import io.inverno.guide.ticket.internal.rest.v1.dto.PlanDto;
import io.inverno.guide.ticket.internal.service.PlanService;
import io.inverno.mod.base.resource.MediaTypes;
import io.inverno.mod.http.base.Method;
import io.inverno.mod.http.base.NotFoundException;
import io.inverno.mod.http.base.Status;
import io.inverno.mod.http.base.header.Headers;
import io.inverno.mod.web.WebExchange;
import io.inverno.mod.web.annotation.*;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

import java.util.List;
import java.util.Optional;

/**
 * Create, update and delete Plans and links tickets to Plans.
 */
@Bean( visibility = Bean.Visibility.PRIVATE )
@WebController( path = "/api/v1/plan" )
public class PlanWebController {

    private final PlanService planService;
    private final DtoMapper<PlanDto, Plan> planDtoMapper;

    public PlanWebController(PlanService planService, DtoMapper<PlanDto, Plan> planDtoMapper) {
        this.planService = planService;
        this.planDtoMapper = planDtoMapper;
    }
    
    ...
}
```

The important part here is the definition of a root path `/api/v1/plan` in the `@WebController` annotation which means that all routes defined in the controller will be relative to `/api/v1/plan` URI.

The plan controller basically exposed the method defined in the `PlanService` as Web routes, this is pretty straightforward but let's see each of them in details to properly understand how Web routes are defined in a Web controller.

The `createPlan()` route handler method delegates the creation of a plan to the `PlanService` and must return `201` response with the new plan id in the `location` HTTP header.

```java
package io.inverno.guide.ticket.internal.rest.v1;

...

@Bean( visibility = Bean.Visibility.PRIVATE )
@WebController( path = "/api/v1/plan" )
public class PlanWebController {
 
    ...

    /**
     * Create a new plan.
     *
     * @param plan     the plan to create
     * @param exchange
     *
     * @return {@inverno.web.status 201} the created plan
     */
    @WebRoute(method = Method.POST, consumes = MediaTypes.APPLICATION_JSON, produces = MediaTypes.APPLICATION_JSON)
    public Mono<PlanDto> createPlan(@Body PlanDto plan, WebExchange<?> exchange) {
        plan.setId(null);
        return this.planDtoMapper.toDomain(plan)
                .flatMap(this.planService::savePlan)
                .doOnNext(savedPlan -> 
                    exchange.response().headers(headers -> headers
                        .status(Status.CREATED)
                        .add(Headers.NAME_LOCATION, exchange.request().getPathBuilder().segment(savedPlan.getId().toString()).buildPath())
                    )
                )
                .flatMap(this.planDtoMapper::toDto);
    }
}
```

A Web route is defined as a regular method annotated with `@io.inverno.mod.web.annotation.WebRoute` whose arguments are bound to the request (body, parameters, headers, cookies...). The method's return value is bound to the response body and thrown exceptions represent error responses.

The `@WebRoute` annotation specifies routing information used to configure the Web router that routes HTTP requests to the right handler, here the `createPlan()` method which is invoked when receiving a `POST` HTTP request targeting `/api/v1/plan` (since no path is defined, the root path defined at Web controller level is used) with an `application/json` content type and accepting `application/json` in response body. Specifying `consumes` and `produces` are actually very important since it also tells the Web router which converters should be used to respectively deserialize and serialize request and response bodies.

The `@Body` annotation on the `plan` method argument indicates that the route expects a request body that can be deserialized to a `PlanDto` object, the optional `WebExchange<?>` argument is the underlying HTTP exchange (request/response pair), it is injected when the method is invoked and allows specifying HTTP headers or the HTTP status (other than `200` which is the default when no error is thrown) in the response and accessing the path builder used to build the path to the newly created resource.

As you can see, all parts of the application are reactive and can be easily composed to implement complex logic fluently. In above implementation, the DTO is first mapped to a Domain object, then the plan service is invoked to save the plan, response status and headers are set in the response on success, the new plan is then mapped to a DTO and eventually returned. All this is done in a concise and efficient way.

Behind the scene the request body is automatically deserialized from JSON to a `PlanDto` object and the response body automatically serialized to JSON following the route definition.

If you looked closely to the JavaDoc, you might have noted the `{@inverno.web.status 201}` custom tag which indicates the status code returned in a successful HTTP response. This tag will be parsed by the Inverno Web compiler when generating the OpenAPI specification. 

The `listPlans()` route handler method delegates to the `PlanService` to list the plans, it then sets the associated tickets to null to remove them from the resulting plans since this method should only list the plans without resolving tickets.

> You should remember that the list of tickets associated to a plan is defined as a `Flux<Ticket>`, as a result tickets are only retrieved when this publisher is subscribed, setting the `tickets` field to null simply tells the DTO mapper to ignore that field.

```java
package io.inverno.guide.ticket.internal.rest.v1;

...

@Bean( visibility = Bean.Visibility.PRIVATE )
@WebController( path = "/api/v1/plan" )
public class PlanWebController {
 
    ...

    /**
     * List plans.
     *
     * @return the list of plans
     */
    @WebRoute( method = Method.GET, produces = MediaTypes.APPLICATION_JSON )
    public Flux<PlanDto> listPlans() {
        return this.planService.listPlans()
            .doOnNext(plan -> plan.setTickets(null)) // We don't want to return tickets when listing plans
            .flatMap(this.planDtoMapper::toDto);
    }
}
```

The `getPlan()` route handler method returns a plan with the list of associated tickets filtered by statuses.

```java
package io.inverno.guide.ticket.internal.rest.v1;

...

@Bean( visibility = Bean.Visibility.PRIVATE )
@WebController( path = "/api/v1/plan" )
public class PlanWebController {
 
    ...

    /**
     * Get a plan by id with its associated tickets filtered by status.
     *
     * @param planId   the id of the plan to get
     * @param statuses the statuses of the tickets to include, if not specified include all tickets
     *
     * @return a plan
     * @throws NotFoundException if there's no plan with the specified id
     */
    @WebRoute( path = "/{planId}", method = Method.GET, produces = MediaTypes.APPLICATION_JSON )
    public Mono<PlanDto> getPlan(@PathParam long planId, @QueryParam Optional<List<Ticket.Status>> statuses) {
        return statuses.map(s -> this.planService.getPlan(planId, s)).orElse(this.planService.getPlan(planId))
            .flatMap(this.planDtoMapper::toDto)
            .switchIfEmpty(Mono.error(() -> new NotFoundException()));
    }
}
```

A path `/{planId}` is defined in the route, it is relative to the root path which is defined in the Web controller, so in order to get a plan, the HTTP request must target `/api/v1/plan/{planId}` where `{planId}` is the id of the plan to get. A path parameter specified between curly braces `{}` must match a method argument annotated with `@io.inverno.mod.web.annotation.PathParam`, here it is bound to `planId`.

An optional query parameter named after the method argument `statuses` annotated with `@io.inverno.mod.web.annotation.QueryParam` is also defined. This parameter is of type `Optional<List<Ticket.Status>>` which indicates that it is not required to invoke the route and that the parameter value, if present, must be converted to a list of `Ticket.Status`. The conversion is done automatically using a parameter converter which converts comma-separated lists of strings to lists of enums.

A `404` HTTP response is returned when no ticket exists with the specified plan id. In such situation, the plan service returns an empty `Mono` which can be switched to an error `Mono` raising a `NotFoundException`. This exception extends `HttpException` which is handled by the Web Error router that eventually returns a `404` HTTP response.

The `@throws` tag in the JavaDoc is used by the Inverno Web compiler to document the `404` HTTP response when generating the OpenAPI specification.

The `updatePlan()` and `deletePlan()` route handler methods are based on what you've just seen. 

```java
package io.inverno.guide.ticket.internal.rest.v1;

...

@Bean( visibility = Bean.Visibility.PRIVATE )
@WebController( path = "/api/v1/plan" )
public class PlanWebController {

    ...

    /**
     * Update a plan.
     *
     * @param planId the id of the plan to update
     * @param plan   the updated plan
     *
     * @return the updated plan
     * @throws NotFoundException if there's no plan with the specified id
     */
    @WebRoute( path = "/{planId}", method = Method.PUT, consumes = MediaTypes.APPLICATION_JSON, produces = MediaTypes.APPLICATION_JSON )
    public Mono<PlanDto> updatePlan(@PathParam long planId, @Body PlanDto plan) {
        plan.setId(planId);
        return this.planDtoMapper.toDomain(plan)
            .flatMap(this.planService::savePlan)
            .flatMap(this.planDtoMapper::toDto)
            .switchIfEmpty(Mono.error(() -> new NotFoundException()));
    }

    /**
     * Delete a plan.
     *
     * @param planId the id of the plan to delete
     *
     * @return the deleted plan
     * @throws NotFoundException if there's no plan with the specified id
     */
    @WebRoute( path = "/{planId}", method = Method.DELETE, produces = MediaTypes.APPLICATION_JSON )
    public Mono<PlanDto> deletePlan(@PathParam long planId) {
        return this.planService.removePlan(planId)
            .flatMap(this.planDtoMapper::toDto)
            .switchIfEmpty(Mono.error(() -> new NotFoundException()));
    }
}
```

The `pushTicket()` route handler method is more interesting as the Web route consumes `application/x-www-form-urlencoded` with multiple form parameters.

```java
package io.inverno.guide.ticket.internal.rest.v1;

...

@Bean( visibility = Bean.Visibility.PRIVATE )
@WebController( path = "/api/v1/plan" )
public class PlanWebController {

    ...

    /**
     * Add a ticket to a plan.
     *
     * @param planId            the id of the plan
     * @param ticketId          the id of the ticket to add
     * @param referenceTicketId the id of the reference ticket before which the ticket must be added, if not specified add the ticket at the end of the list
     *
     * @return
     */
    @WebRoute( path = "/{planId}/ticket", method = Method.POST, consumes= MediaTypes.APPLICATION_X_WWW_FORM_URLENCODED )
    public Mono<Void> pushTicket(@PathParam long planId, @FormParam long ticketId, @FormParam Optional<Long> referenceTicketId) {
        return referenceTicketId
            .map(refTicketId -> this.planService.insertTicketBefore(planId, ticketId, refTicketId))
            .orElse(this.planService.addTicket(planId, ticketId));
    }
}
```

In addition to the `planId` path parameter, there are two form parameters named after the method arguments annotated with `@io.inverno.mod.web.annotation.FormParameter`. Unlike the `referenceTicketId` parameter which is optional and not required to invoke the route, the `ticketId` parameter is required and a `MissingRequiredParameterException`, resulting in a `400` HTTP response, will be raised if it is missing from the request.

Finally, the `removeTicket()` route handler achieves the `PlanWebController` class.

```java
package io.inverno.guide.ticket.internal.rest.v1;

...

@Bean( visibility = Bean.Visibility.PRIVATE )
@WebController( path = "/api/v1/plan" )
public class PlanWebController {

    ...

    /**
     * Remove a ticket from a plan.
     *
     * @param planId   the id of the plan
     * @param ticketId the id of the ticket to remove
     *
     * @return 1 if the ticket was removed, 0 if the ticket wasn't associated to the plan
     */
    @WebRoute( path = "/{planId}/ticket/{ticketId}", method = Method.DELETE, produces = MediaTypes.TEXT_PLAIN )
    public Mono<Long> removeTicket(@PathParam long planId, @PathParam long ticketId) {
        return this.planService.removeTicket(planId, ticketId);
    }
}
```

The `TicketWebController` is implemented in a similar way, it exposes tickets and ticket notes as a result it requires `TicketService`, `NoteService`, `DtoMapper<TicketDto, Ticket>` and `DtoMapper<NoteDto, Note>` instances.

```java
package io.inverno.guide.ticket.internal.rest.v1;

import io.inverno.core.annotation.Bean;
import io.inverno.guide.ticket.internal.model.Note;
import io.inverno.guide.ticket.internal.model.Ticket;
import io.inverno.guide.ticket.internal.rest.DtoMapper;
import io.inverno.guide.ticket.internal.rest.v1.dto.NoteDto;
import io.inverno.guide.ticket.internal.rest.v1.dto.TicketDto;
import io.inverno.guide.ticket.internal.service.NoteService;
import io.inverno.guide.ticket.internal.service.TicketService;
import io.inverno.mod.base.resource.MediaTypes;
import io.inverno.mod.http.base.Method;
import io.inverno.mod.http.base.NotFoundException;
import io.inverno.mod.http.base.Status;
import io.inverno.mod.http.base.header.Headers;
import io.inverno.mod.web.WebExchange;
import io.inverno.mod.web.annotation.*;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

import java.util.List;
import java.util.Optional;

/**
 * Create, update and delete Tickets and manages tickets notes.
 */
@Bean( visibility = Bean.Visibility.PRIVATE )
@WebController( path = "/api/v1/ticket" )
public class TicketWebController {

    private final TicketService ticketService;
    private final NoteService noteService;

    private final DtoMapper<TicketDto, Ticket> ticketDtoMapper;
    private final DtoMapper<NoteDto, Note> noteDtoMapper;

    public TicketWebController(TicketService ticketService, NoteService noteService, DtoMapper<TicketDto, Ticket> ticketDtoMapper, DtoMapper<NoteDto, Note> noteDtoMapper) {
        this.ticketService = ticketService;
        this.noteService = noteService;
        this.ticketDtoMapper = ticketDtoMapper;
        this.noteDtoMapper = noteDtoMapper;
    }

    /** Create a next ticket. ...*/
    @WebRoute( method = Method.POST, consumes = MediaTypes.APPLICATION_JSON, produces = MediaTypes.APPLICATION_JSON )
    public Mono<TicketDto> createTicket(@Body TicketDto ticket, WebExchange<?> exchange) {...}

    /** List tickets. ...*/
    @WebRoute( method = Method.GET, produces = MediaTypes.APPLICATION_JSON )
    public Flux<TicketDto> listTickets(@QueryParam Optional<List<Ticket.Status>> statuses) {...}

    /** Get a ticket by id. ...*/
    @WebRoute( path = "/{ticketId}", method = Method.GET, produces = MediaTypes.APPLICATION_JSON )
    public Mono<TicketDto> getTicket(@PathParam long ticketId) {...}

    /** Update a ticket. ...*/
    @WebRoute( path = "/{ticketId}", method = Method.PUT, consumes = MediaTypes.APPLICATION_JSON, produces = MediaTypes.APPLICATION_JSON )
    public Mono<TicketDto> updateTicket(@PathParam long ticketId, @Body TicketDto ticket) {...}

    /** Update the status of a ticket. ...*/
    @WebRoute( path = "/{ticketId}/status", method = Method.POST, consumes = MediaTypes.TEXT_PLAIN, produces = MediaTypes.APPLICATION_JSON)
    public Mono<TicketDto> updateTicketStatus(@PathParam long ticketId, @Body Ticket.Status status) {...}

    /** Delete a ticket. ...*/
    @WebRoute( path = "/{ticketId}", method = Method.DELETE, produces = MediaTypes.APPLICATION_JSON )
    public Mono<TicketDto> deleteTicket(@PathParam long ticketId) {...}

    /** Create a ticket note. ...*/
    @WebRoute( path = "/{ticketId}/note", method = Method.POST, consumes = MediaTypes.APPLICATION_JSON, produces = MediaTypes.APPLICATION_JSON )
    public Mono<NoteDto> createTicketNote(@PathParam long ticketId, @Body NoteDto note, WebExchange<?> exchange) {...}

    /** List notes associated to a ticket. ...*/
    @WebRoute( path = "/{ticketId}/note", method = Method.GET, produces = MediaTypes.APPLICATION_JSON )
    public Flux<NoteDto> listTicketNotes(@PathParam long ticketId) {...}

    /** Get a ticket note. ...*/
    @WebRoute( path = "/{ticketId}/note/{noteIndex}", method = Method.GET, produces = MediaTypes.APPLICATION_JSON )
    public Mono<NoteDto> getTicketNote(@PathParam long ticketId, @PathParam int noteIndex) {...}

    /** Update a ticket note. ...*/
    @WebRoute( path = "/{ticketId}/note/{noteIndex}", method = Method.PUT, consumes = MediaTypes.APPLICATION_JSON, produces = MediaTypes.APPLICATION_JSON )
    public Mono<NoteDto> updateTicketNote(@PathParam long ticketId, @PathParam int noteIndex, @Body NoteDto note) {...}

    /** Delete a ticket note. ...*/
    @WebRoute( path = "/{ticketId}/note/{noteIndex}", method = Method.DELETE, produces = MediaTypes.APPLICATION_JSON )
    public Mono<NoteDto> deleteTicketNote(@PathParam long ticketId, @PathParam int noteIndex) {...}
}
```

The complete code can be found in [GitHub](https://github.com/inverno-io/inverno-apps/blob/1.1.0/inverno-ticket/src/main/java/io/inverno/app/ticket/internal/rest/v1/TicketWebController.java)

As for the Domain model, you must add the following `exports` directive to the module descriptor to allow Jackson to access DTOs.

```java
@io.inverno.core.annotation.Module
module io.inverno.guide.ticket {
    requires io.inverno.mod.boot;
    requires io.inverno.mod.redis.lettuce;

    exports io.inverno.guide.ticket.internal.model to com.fasterxml.jackson.databind;
    exports io.inverno.app.ticket.internal.rest.v1.dto to com.fasterxml.jackson.databind;
}
```

The Inverno Web compiler does not generate OpenAPI specifications by default, this generation must be activated explicitly in the Maven compiler plugin's configuration in the Maven project descriptor.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>io.inverno.dist</groupId>
        <artifactId>inverno-parent</artifactId>
        <version>${VERSION_INVERNO_DIST}</version>
    </parent>
    <groupId>io.inverno.guide</groupId>
    <artifactId>ticket</artifactId>
    <version>1.0-SNAPSHOT</version>

    ...
    
    <build>
        <pluginManagement>
            <plugins>
                ...
                <plugin>
                    <groupId>org.apache.maven.plugins</groupId>
                    <artifactId>maven-compiler-plugin</artifactId>
                    <configuration>
                        <compilerArgs>
                            <arg>--module-version=${project.version}</arg>
                            <arg>-Ainverno.web.generateOpenApiDefinition=true</arg>
                        </compilerArgs>
                    </configuration>
                </plugin>
            </plugins>
        </pluginManagement>
    </build>
</project>
```

You are now ready to test the application, but first you need to start a Redis data store. This is pretty easy using Docker:

```text
$ docker run -d -p6379:6379 redis
```

The Inverno Ticket application is run as follows:

```text
$ mvn inverno:run
...
[INFO] --- inverno-maven-plugin:${VERSION_INVERNO_TOOLS}:run (default-cli) @ ticket ---
[INFO] Running project: io.inverno.guide.ticket@1.0-SNAPSHOT...
ERROR StatusLogger Log4j2 could not find a logging implementation. Please add log4j-core to the classpath. Using SimpleLogger to log to the console...
INFO Application Inverno is starting...


     ╔════════════════════════════════════════════════════════════════════════════════════════════╗
     ║                      , ~~ ,                                                                ║
     ║                  , '   /\   ' ,                                                            ║
     ║                 , __   \/   __ ,      _                                                    ║
     ║                ,  \_\_\/\/_/_/  ,    | |  ___  _    _  ___   __  ___   ___                 ║
     ║                ,    _\_\/_/_    ,    | | / _ \\ \  / // _ \ / _|/ _ \ / _ \                ║
     ║                ,   __\_/\_\__   ,    | || | | |\ \/ /|  __/| | | | | | |_| |               ║
     ║                 , /_/ /\/\ \_\ ,     |_||_| |_| \__/  \___||_| |_| |_|\___/                ║
     ║                  ,     /\     ,                                                            ║
     ║                    ,   \/   ,                                 -- ${VERSION_INVERNO_CORE} --                  ║
     ║                      ' -- '                                                                ║
     ╠════════════════════════════════════════════════════════════════════════════════════════════╣
     ║ Java runtime        : OpenJDK Runtime Environment                                          ║
     ║ Java version        : 17+35-2724                                                           ║
     ║ Java home           : /home/jkuhn/Devel/jdk/jdk-17                                         ║
     ║                                                                                            ║
     ║ Application module  : io.inverno.guide.ticket                                              ║
     ║ Application version : 1.0-SNAPSHOT                                                         ║
     ║ Application class   : io.inverno.guide.ticket.App                                          ║
     ║                                                                                            ║
     ║ Modules             :                                                                      ║
     ║  * ...                                                                                     ║
     ╚════════════════════════════════════════════════════════════════════════════════════════════╝


INFO Ticket Starting Module io.inverno.guide.ticket...
INFO Boot Starting Module io.inverno.mod.boot...
INFO Boot Module io.inverno.mod.boot started in 337ms
INFO Lettuce Starting Module io.inverno.mod.redis.lettuce...
INFO Lettuce Module io.inverno.mod.redis.lettuce started in 44ms
INFO Web Starting Module io.inverno.mod.web...
INFO Server Starting Module io.inverno.mod.http.server...
INFO Base Starting Module io.inverno.mod.http.base...
INFO Base Module io.inverno.mod.http.base started in 4ms
INFO HttpServer HTTP Server (nio) listening on http://0.0.0.0:8080
INFO Server Module io.inverno.mod.http.server started in 95ms
INFO Web Module io.inverno.mod.web started in 95ms
INFO Ticket Module io.inverno.guide.ticket started in 480ms
INFO Application Application io.inverno.guide.ticket started in 554ms
```

You can test the REST API:

```text
$ curl -i -X POST -H 'content-type: application/json' -d '{"title":"My first plan", "summary":"This is my first plan", "description":"Lorem ipsum dolor sit amet"}' http://localhost:8080/api/v1/plan
HTTP/1.1 201 Created
content-type: application/json
location: /api/v1/plan/1
content-length: 174

{"id":1,"title":"My first plan","summary":"This is my first plan","description":"Lorem ipsum dolor sit amet","creationDateTime":"2022-02-23T08:46:02.536865752Z","tickets":[]}

$ curl -i http://localhost:8080/api/v1/plan
HTTP/1.1 200 OK
content-type: application/json
transfer-encoding: chunked

[{"id":1,"title":"My first plan","summary":"This is my first plan","description":"Lorem ipsum dolor sit amet","creationDateTime":"2022-02-23T08:46:02.536865752Z","tickets":[]}]

$ curl -i http://localhost:8080/api/v1/plan/1
HTTP/1.1 200 OK
content-type: application/json
content-length: 174

{"id":1,"title":"My first plan","summary":"This is my first plan","description":"Lorem ipsum dolor sit amet","creationDateTime":"2022-02-23T08:46:02.536865752Z","tickets":[]}
```

An OpenAPI specification should have been generated in the project build output directory: `./target/classes/META-INF/inverno/web/io.inverno.guide.ticket/openapi.yml`.

## Step 6: Create the Front-end layer

The Front-end layer is composed of the *static* resources of the application including the application Web UI which is a Single-page application, all its dependencies, the OpenAPI specifications and a [Swagger UI](https://swagger.io/tools/swagger-ui/) to visualize and interact with the application's REST API. All these resources are served by the application.

The application Web UI is a Single-page application consuming the REST API and built with [Bootstrap](https://getbootstrap.com/) and [Vue.js](https://v3.vuejs.org/). The UI resources of the application should be placed in a `static/` directory in application module's resources folder `src/main/resources`. You must create the following file structure:

```text
src/main/resources/static/
├── css
├── img
├── js
└── index.html
```

Web UI development using Bootstrap and Vue.js is not the purpose of this guide, so please refer to appropriate documentations if you want to go deeper. The complete UI code can be found in [GitHub](https://github.com/inverno-io/inverno-apps/tree/1.1.0/inverno-ticket/src/main/resources/static).

The Web UI requires multiple JavaScript libraries that are packaged as [WebJars](https://www.webjars.org/), you'll also need Swagger UI resources which also comes as a WebJar. These are quite easy to include into the application by declaring corresponding dependencies in the Maven project descriptor:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>io.inverno.dist</groupId>
        <artifactId>inverno-parent</artifactId>
        <version>${VERSION_INVERNO_DIST}</version>
    </parent>
    <groupId>io.inverno.guide</groupId>
    <artifactId>ticket</artifactId>
    <version>1.0-SNAPSHOT</version>

    <dependencies>
        ...

        <dependency>
            <groupId>org.webjars.npm</groupId>
            <artifactId>vue</artifactId>
            <version>3.2.26</version>
        </dependency>
        <dependency>
            <groupId>org.webjars</groupId>
            <artifactId>bootstrap</artifactId>
            <version>5.1.3</version>
        </dependency>
        <dependency>
            <groupId>org.webjars.npm</groupId>
            <artifactId>bootstrap-icons</artifactId>
            <version>1.7.2</version>
        </dependency>
        <dependency>
            <groupId>org.webjars.npm</groupId>
            <artifactId>marked</artifactId>
            <version>4.0.8</version>
        </dependency>
        <dependency>
            <groupId>org.webjars</groupId>
            <artifactId>highlightjs</artifactId>
            <version>10.1.2</version>
        </dependency>
        <dependency>
            <groupId>org.webjars</groupId>
            <artifactId>swagger-ui</artifactId>
        </dependency>
    </dependencies>

    ...
</project>
```

> The Swagger UI artifact is managed by Inverno's distribution, so you don't need to specify its version.

You must now configure the application's Web router by defining appropriate Web routes to serve all these resources. The Inverno Web module provides built-in configurers and route handlers to easily expose WebJars and any kind of static resources. Let's create a `StaticWebRoutesConfigurer` that will let you programmatically configure the Web router.

```java
package io.inverno.guide.ticket.internal;

import io.inverno.core.annotation.Bean;
import io.inverno.guide.ticket.AppConfiguration;
import io.inverno.mod.base.resource.Resource;
import io.inverno.mod.base.resource.ResourceService;
import io.inverno.mod.http.base.Method;
import io.inverno.mod.http.server.ExchangeContext;
import io.inverno.mod.web.*;

@Bean(visibility = Bean.Visibility.PRIVATE)
public class StaticWebRoutesConfigurer implements WebRoutesConfigurer<ExchangeContext> {

    private final AppConfiguration configuration;
    private final ResourceService resourceService;
    private final Resource homeResource;

    public StaticWebRoutesConfigurer(AppConfiguration configuration, ResourceService resourceService) {
        this.configuration = configuration;
        this.resourceService = resourceService;
        this.homeResource = this.resourceService.getResource(this.configuration.web_root()).resolve("index.html");
    }

    @Override
    public void accept(WebRoutable<ExchangeContext, ?> routes) {
        routes
            // OpenAPI specifications
            .configureRoutes(new OpenApiRoutesConfigurer<>(this.resourceService, true))
            // WebJars
            .configureRoutes(new WebJarsRoutesConfigurer<>(this.resourceService))
            // Static resources: html, javascript, css, images...
            .route()
                .path("/static/{path:.*}", true)
                .method(Method.GET)
                .handler(new StaticHandler<>(this.resourceService.getResource(this.configuration.web_root())))
            // Welcome page
            .route()
                .path("/", true)
                .method(Method.GET)
                .handler(exchange -> exchange.response().body().resource().value(this.homeResource))

    }
}
```

The `StaticWebRoutesConfigurer` class is an implementation of `WebRoutesConfigurer` which is used to configure routes in the Web router. It is injected into the Web router when the application module is started.

The HTTP server root directory, which basically points to the application Web UI resources, must be made configurable by defining a `web_root` configuration property in the `AppConfiguration`.

```java
package io.inverno.guide.ticket;

import io.inverno.core.annotation.NestedBean;
import io.inverno.mod.boot.BootConfiguration;
import io.inverno.mod.configuration.Configuration;
import io.inverno.mod.redis.lettuce.LettuceRedisClientConfiguration;
import io.inverno.mod.web.WebConfiguration;

import java.net.URI;

@Configuration
public interface AppConfiguration {

    ...

    default URI web_root() {
        return URI.create("module://" + AppConfiguration.class.getModule().getName() + "/static");
    }
}
```

The `web_root` property is a resource URI that is passed to the `ResourceService`. The `ResourceService` is provided by the Inverno boot module, it provides unified access to resources based on URIs. For instance, it can be used to resolve file resources (`file:/...`), class path resources (`classpath:/...`), resources inside JAR or ZIP files (`jar:/...`), network resources (`http://...`, `ftp://...`) or module resources (`module:/...`). In above code, the root directory points to the `static/` directory inside the application module by default.

The `homeResource` is resolved once and represents the welcome page, namely the `static/index.html` page used to bootstrap the application UI.

Routes are configured in the `accept()` method, the `routable` argument of type `WebRoutable` allows defining routes in a fluent way, similar to what you saw with Web controllers. 

The `OpenApiRoutesConfigurer` and `WebJarsRoutesConfigurer` are built-in routes configurers, used respectively to configure Web routes to generated OpenAPI specifications, with or without Swagger UI, and to configure Web routes to WebJars present on the class path or the module path.

Static resources under `web_root` are mapped to `/static/{path:.*}` path using a `StaticHandler` which serves any resources under `web_root` where `path` path parameter is the relative path to the resource under `web_root`.

Finally, the `homeResource` is explicitly mapped to the root path `/` .

If you rebuild the application and open [http://localhost:8080](http://localhost:8080) in your Web browser, you should see the Inverno Ticket application UI.

```text
$ mvn clean inverno:run
...
```

<img class="shadow" src="img/inverno_ticket_app_0.png" style="display: block; margin: 2em auto;" alt="Inverno Ticket Application"/>

You can also display a fully functional Swagger UI exposing the generated OpenAPI specification at [http://localhost:8080/open-api](http://localhost:8080/open-api).

<img class="shadow" src="img/inverno_ticket_rest_api.png" style="display: block; margin: 2em auto;" alt="Inverno Ticket REST API"/>

You can play a bit with the application (or the Swagger UI) by creating a plan and some tickets to validate that everything is working fine.

<img class="shadow" src="img/inverno_ticket_app_1.png" style="display: block; margin: 2em auto;" alt="Inverno Full Stack Guide plan"/>

Now if you try to modify the UI code, you won't be able to see changes live. This is because the Inverno Maven plugin modularizes and packages project dependencies before running the application as a result the `src/main/resources/static/` folder is packaged within the runtime module, since the `web_root` configuration property targets this location by default, changes in `src/main/resources/static/` directory are not loaded unless the project is rebuilt. 

This is clearly not convenient when developing Web UIs, hopefully the `web_root` configuration property can be configured on the command line as follows:

```text
$ mvn inverno:run -Dinverno.run.arguments="--io.inverno.guide.ticket.appConfiguration.web_root=\\\"file:/path/to/project/src/main/resources/static\\\""
...
```

You might also prefer pointing to the local source directory during development and otherwise to the module's location, which is the default behaviour. The Inverno configuration API supports parameterized configuration which allows defining different values for a given configuration property based on a set of parameters. In this particular case, you can rely on a `profile` parameter that you can inject when the module is started.

```java
package io.inverno.guide.ticket;

import io.inverno.core.annotation.Bean;
import io.inverno.core.v1.Application;
import io.inverno.mod.configuration.ConfigurationKey;
import io.inverno.mod.configuration.ConfigurationProperty;
import io.inverno.mod.configuration.ConfigurationSource;
import io.inverno.mod.configuration.source.BootstrapConfigurationSource;
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;

import java.io.IOException;
import java.util.List;
import java.util.function.Supplier;

public class App {

    private static final Logger LOGGER = LogManager.getLogger(App.class);

    public static final String REDIS_KEY = "APP:Ticket";
    public static final String PROFILE_PROPERTY_NAME = "profile";

    @Bean( name = "configurationSource")
    public interface AppConfigurationSource extends Supplier<ConfigurationSource<?, ?, ?>> {}

    @Bean( name = "configurationParameters")
    public static interface TicketAppConfigurationParameters extends Supplier<List<ConfigurationKey.Parameter>> {}

    public static void main( String[] args ) throws IOException {
        final BootstrapConfigurationSource bootstrapConfigurationSource = new BootstrapConfigurationSource(App.class.getModule(), args);
        bootstrapConfigurationSource
                .get(PROFILE_PROPERTY_NAME)
                .execute()
                .single()
                .map(configurationQueryResult -> configurationQueryResult.getResult().flatMap(ConfigurationProperty::asString).orElse("default"))
                .map(profile -> {
                    LOGGER.info(() -> "Active profile: " + profile);
                    return Application.run(new Ticket.Builder()
                            .setConfigurationSource(bootstrapConfigurationSource)
                            .setConfigurationParameters(List.of(ConfigurationKey.Parameter.of(PROFILE_PROPERTY_NAME, profile)))
                    );
                })
                .block();

    }
}
```

In above code, the `profile` value is first resolved using the bootstrap configuration source, it is then injected into the module by defining `configurationParameters` socket bean. Using the bootstrap configuration source to resolve the profile parameter has many advantages, for instance you can define the profile as an environment variable, a system property or a command line argument. The bootstrap configuration source also support defaulting: command line arguments override system properties which override environment variables... 

You can now specify a `dev` location for the `web_root` configuration property in `src/main/resources/configuration.cprops`

```text
io.inverno.guide.ticket.appConfiguration {
    [ profile = "dev" ] {
        web_root = "file:/path/to/project/src/main/resources/static"
    }
}
```

If you restart the application with command line argument `--profile=\"dev\"`, you should be able to modify Web UI resources and see changes live.

```text
$ mvn inverno:run -Dinverno.run.arguments="--profile=\\\"dev\\\""
...
14:58:35.258 [main] INFO  io.inverno.guide.ticket.App - Active profile: dev
14:58:35.468 [main] INFO  io.inverno.core.v1.Application - Inverno is starting...
...
```

> You might wonder why quotes must be escaped when specifying command line arguments values. This is because configuration values are typed and a String value must be specified following the Java String syntax, since quotes might be interpreted by the shell, they have to be escaped and even double escaped when specified in a system property (i.e. `-Dinverno.run.arguments="..."`).

## Step 7: Configure Logging

Inverno relies on [Apache Log4j2](https://logging.apache.org/log4j/2.x/) for logging. So far, the Log4j2 runtime wasn't included and Log4j default `SimpleLogger` implementation was used. In order to provide a more advanced logging configuration, you need to declare `log4j-core` dependency in the Maven project descriptor.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>io.inverno.dist</groupId>
        <artifactId>inverno-parent</artifactId>
        <version>${VERSION_INVERNO_DIST}</version>
    </parent>
    <groupId>io.inverno.guide</groupId>
    <artifactId>ticket</artifactId>
    <version>1.0-SNAPSHOT</version>

    <dependencies>
        ...

        <dependency>
            <groupId>org.apache.logging.log4j</groupId>
            <artifactId>log4j-core</artifactId>
        </dependency>
    </dependencies>

    ...
</project>
```

At this stage, you might also want to declare a dependency to the Log4j2 API module in the Java module descriptor as you'd probably want to add some logs in your application.

```java
@io.inverno.core.annotation.Module
module io.inverno.guide.ticket {
    requires io.inverno.mod.boot;
    requires io.inverno.mod.redis.lettuce;
    requires io.inverno.mod.web;

    requires org.apache.logging.log4j;

    exports io.inverno.guide.ticket.internal.model to com.fasterxml.jackson.databind;
    exports io.inverno.guide.ticket.internal.rest.v1.dto to com.fasterxml.jackson.databind;
}
```

Since Inverno Ticket Application should be production-ready, logging must be configured to log application logs, access logs and errors logs in separate rolling files. The application might eventually run in the cloud, in an Amazon EC2 instance for example, so let's also format logs in such a way that they can be easily integrated with tools like [Amazon CloudWatch](https://aws.amazon.com/cloudwatch/).

In order to format logs in the JSON formats expected by AWS, you must add `log4j-layout-template-json` dependency to the Maven project descriptor. As for Lettuce library, Log4j hasn't been migrated to a Java module yet, so you'll also need to set some VM options in the Inverno plugin configuration to avoid runtime errors:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>io.inverno.dist</groupId>
        <artifactId>inverno-parent</artifactId>
        <version>${VERSION_INVERNO_DIST}</version>
    </parent>
    <groupId>io.inverno.guide</groupId>
    <artifactId>ticket</artifactId>
    <version>1.0-SNAPSHOT</version>

    <dependencies>
        ...

        <dependency>
            <groupId>org.apache.logging.log4j</groupId>
            <artifactId>log4j-core</artifactId>
        </dependency>
        <dependency>
            <groupId>org.apache.logging.log4j</groupId>
            <artifactId>log4j-layout-template-json</artifactId>
        </dependency>
    </dependencies>

    <build>
        <pluginManagement>
            <plugins>
                <plugin>
                    <groupId>io.inverno.tool</groupId>
                    <artifactId>inverno-maven-plugin</artifactId>
                    <configuration>
                        <vmOptions>--add-opens reactor.core/reactor.core.publisher=lettuce.core --add-opens org.apache.logging.log4j.core/org.apache.logging.log4j.core.jackson=com.fasterxml.jackson.databind --add-opens org.apache.logging.log4j.log4j.layout.template.json/org.apache.logging.log4j.layout.template.json=org.apache.logging.log4j.core</vmOptions>
                    </configuration>
                </plugin>
                ...
            </plugins>
        </pluginManagement>
    </build>
</project>
```

The access log layout expected by AWS is a bit specific and requires to define a custom `AccessLayout.json` in project resources `src/main/resources`:

```json
{
    "@timestamp": {
        "$resolver": "timestamp",
        "pattern": {
            "format": "yyyy-MM-dd'T'HH:mm:ss.SSS'Z'",
            "timeZone": "UTC"
        }
    },
    "message": {
        "$resolver": "message",
        "stringified": false
    }
}
```

Log4j can be configured in `log4j2.xml` file in project resources folder `src/main/resources`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="WARN" name="Website" shutdownHook="disable">
    <Appenders>
        <Console name="Console" target="SYSTEM_OUT">
            <PatternLayout pattern="%d{DEFAULT} %highlight{%-5level} [%t] %c{1.} - %msg%n%ex"/>
        </Console>
        <!-- Application log -->
        <RollingRandomAccessFile name="ApplicationRollingFile" fileName="logs/application.log" filePattern="logs/error-%d{yyyy-MM-dd}-%i.log.gz">
            <JsonTemplateLayout/>
            <Policies>
                <TimeBasedTriggeringPolicy />
                <SizeBasedTriggeringPolicy size="10 MB"/>
            </Policies>
            <DefaultRolloverStrategy>
                <Delete basePath="logs" maxDepth="2">
                    <IfFileName glob="application-*.log.gz" />
                    <IfLastModified age="10d" />
                </Delete>
            </DefaultRolloverStrategy>
        </RollingRandomAccessFile>
        <Async name="AsyncApplicationRollingFile">
            <AppenderRef ref="ApplicationRollingFile"/>
        </Async>
        <!-- Error log -->
        <RollingRandomAccessFile name="ErrorRollingFile" fileName="logs/error.log" filePattern="logs/error-%d{yyyy-MM-dd}-%i.log.gz">
            <JsonTemplateLayout/>
            <NoMarkerFilter onMatch="ACCEPT" onMismatch="DENY"/>
            <Policies>
                <TimeBasedTriggeringPolicy />
                <SizeBasedTriggeringPolicy size="10 MB"/>
            </Policies>
            <DefaultRolloverStrategy>
                <Delete basePath="logs" maxDepth="2">
                    <IfFileName glob="error-*.log.gz" />
                    <IfLastModified age="10d" />
                </Delete>
            </DefaultRolloverStrategy>
        </RollingRandomAccessFile>
        <Async name="AsyncErrorRollingFile">
            <AppenderRef ref="ErrorRollingFile"/>
        </Async>
        <!-- Access log -->
        <RollingRandomAccessFile name="AccessRollingFile" fileName="logs/access.log" filePattern="logs/access-%d{yyyy-MM-dd}-%i.log.gz">
            <JsonTemplateLayout eventTemplateUri="classpath:AccessLayout.json"/>
            <MarkerFilter marker="HTTP_ACCESS" onMatch="ACCEPT" onMismatch="DENY"/>
            <Policies>
                <TimeBasedTriggeringPolicy />
                <SizeBasedTriggeringPolicy size="10 MB"/>
            </Policies>
            <DefaultRolloverStrategy>
                <Delete basePath="logs" maxDepth="2">
                    <IfFileName glob="access-*.log.gz" />
                    <IfLastModified age="10d" />
                </Delete>
            </DefaultRolloverStrategy>
        </RollingRandomAccessFile>
        <Async name="AsyncAccessRollingFile">
            <AppenderRef ref="AccessRollingFile"/>
        </Async>
    </Appenders>

    <Loggers>
        <Logger name="io.inverno.mod.http.server.internal.AbstractExchange" additivity="false" level="info">
            <AppenderRef ref="AsyncAccessRollingFile" level="info"/>
            <AppenderRef ref="AsyncErrorRollingFile" level="error"/>
        </Logger>

        <Root level="info" additivity="false">
            <AppenderRef ref="Console" level="info" />
            <AppenderRef ref="ApplicationRollingFile" level="info" />
            <AppenderRef ref="AsyncErrorRollingFile" level="error"/>
        </Root>
    </Loggers>
</Configuration>
{% endraw %}
```

If you restart the application, you should now see three properly formatted log files `application.log`, `access.log` and `error.log` under the `logs/` directory:

```text
$ mvn inverno:run
...

$ ls logs/
access.log  application.log  error.log

$ cat logs/access.log

{"@timestamp":"2022-02-23T15:42:35.441Z","message":{"remoteAddress":"127.0.0.1","request":"GET \/","status":200,"bytes":20708,"referer":"","userAgent":"Mozilla\/5.0 (X11; Linux x86_64; rv:91.0) Gecko\/20100101 Firefox\/91.0"}}
{"@timestamp":"2022-02-23T15:42:35.485Z","message":{"remoteAddress":"127.0.0.1","request":"GET \/webjars\/marked\/marked.min.js","status":200,"bytes":47375,"referer":"http:\/\/localhost:8080\/","userAgent":"Mozilla\/5.0 (X11; Linux x86_64; rv:91.0) Gecko\/20100101 Firefox\/91.0"}}
{"@timestamp":"2022-02-23T15:42:35.492Z","message":{"remoteAddress":"127.0.0.1","request":"GET \/static\/js\/script.js","status":200,"bytes":16743,"referer":"http:\/\/localhost:8080\/","userAgent":"Mozilla\/5.0 (X11; Linux x86_64; rv:91.0) Gecko\/20100101 Firefox\/91.0"}}
...
```

## Step 8: Configure TLS

The Inverno HTTP server can be configured with TLS support (i.e. HTTPS) for secured communications. If you have carefully followed this documentation, activating TLS should come down to creating a server certificate and set a couple of configuration properties.

Let's start by creating a self-signed certificate in project resources folder `src/main/resources/` using `keytool`:

```text
$ keytool -genkey -keyalg RSA -alias selfsigned -keystore keystore.jks -storepass changeit -validity 360 -keysize 2048
...
```

> Do not use self-signed certificate for any other purposes than development and testing.

Since Web module configuration should be already exposed in `AppConfiguration`, you can now configure the HTTP server in `src/main/resources/configuration.cprops`:

```text
io.inverno.guide.ticket.appConfiguration {
    [ profile = "dev" ] {
        web_root = "file:/home/jkuhn/Devel/git/winter/doc/guides/io.inverno.guide.ticket/src/main/resources/static"
    }
    web.http_server {
        server_port = 8443
        tls_enabled = true
        key_store = "module://io.inverno.guide.ticket/keystore.jks"
        key_alias = "selfsigned"
        key_store_password = "changeit"
    }
}
```

If you restart the application, it should now be accessible using HTTPs at [https://localhost:8443](https://localhost:8443).

```text
$ mvn inverno:run
...
2022-02-23 15:33:39,410 INFO  [main] i.i.m.h.s.i.HttpServer - HTTP Server (nio) listening on https://0.0.0.0:8443
...
```

Another interesting thing to notice is that communication is now using HTTP/2 protocol which is activated by default when TLS is configured.

```text
$ curl --insecure -i https://localhost:8443/api/v1/plan
HTTP/2 200 
content-type: application/json

[{"id":1,"title":"Inverno Full Stack Guide","summary":"Develop a Full Stack application with Inverno, Redis and Vue.js","description":null,"creationDateTime":"2022-02-23T13:15:11.411259961Z","tickets":[]}]
```

> Note that HTTP/2 over cleartext (aka H2C) is also supported and can be activated by setting `web.http_server.h2c_enabled` configuration property to true.

You might choose to activate TLS support only on production environment. This can be done using the same approach as for the `web_root` configuration property. Let's modify the configuration to only activate TLS support when the application is started with `prod` profile.

```text
io.inverno.guide.ticket.appConfiguration {
    [ profile = "dev" ] {
        web_root = "file:/home/jkuhn/Devel/git/winter/doc/guides/io.inverno.guide.ticket/src/main/resources/static"
    }
    [ profile = "prod" ] {
        web.http_server {
            server_port = 8443
            tls_enabled = true
            key_store = "module://io.inverno.guide.ticket/keystore.jks"
            key_alias = "selfsigned"
            key_store_password = "changeit"
        }
    }
}
```

Now TLS support should only be activated when the application is started using the `prod` profile:

```text
$ mvn inverno:run -Dinverno.run.arguments="--profile=\\\"prod\\\""
...
2022-02-23 15:46:47,749 INFO  [main] i.i.g.t.App - Active profile: prod
15:46:47.749 [main] INFO  io.inverno.guide.ticket.App - Active profile: prod
...
2022-02-23 15:46:48,776 INFO  [main] i.i.m.h.s.i.HttpServer - HTTP Server (nio) listening on https://0.0.0.0:8443
...

$ mvn inverno:run
...
2022-02-23 15:48:57,222 INFO  [main] i.i.g.t.App - Active profile: default
...
2022-02-23 15:48:57,881 INFO  [main] i.i.m.h.s.i.HttpServer - HTTP Server (nio) listening on http://0.0.0.0:8080
...
```

> The Java keystore is packaged within the application module to keep things simple, however in a real life application, certificates are usually managed externally so the `web.http_server.key_store` should instead point to an external URI (e.g. `file:/path/to/keystore.jks`).

## Step 9: Use native transport

In order to improve performances, the Inverno HTTP server can be configured to used native transport such as [epoll](https://en.wikipedia.org/wiki/Epoll), [kqueue](https://en.wikipedia.org/wiki/Kqueue) or [io_uring](https://en.wikipedia.org/wiki/Io_uring) when the platform supports it.

If you intend to run the application on a Linux system for instance, you can activate epoll native transport by declaring the following dependency in the Maven project descriptor and adding modules `io.netty.transport.unix.common` and `io.netty.transport.epoll` in the Inverno Maven plugin VM options:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>io.inverno.dist</groupId>
        <artifactId>inverno-parent</artifactId>
        <version>${VERSION_INVERNO_DIST}</version>
    </parent>
    <groupId>io.inverno.guide</groupId>
    <artifactId>ticket</artifactId>
    <version>1.0-SNAPSHOT</version>

    <dependencies>
        ...

        <dependency>
            <groupId>io.netty</groupId>
            <artifactId>netty-transport-native-epoll</artifactId>
            <classifier>linux-x86_64</classifier>
        </dependency>
    </dependencies>

    <build>
        <pluginManagement>
            <plugins>
                <plugin>
                    <groupId>io.inverno.tool</groupId>
                    <artifactId>inverno-maven-plugin</artifactId>
                    <configuration>
                        <vmOptions>--add-opens reactor.core/reactor.core.publisher=lettuce.core --add-opens org.apache.logging.log4j.core/org.apache.logging.log4j.core.jackson=com.fasterxml.jackson.databind --add-opens org.apache.logging.log4j.log4j.layout.template.json/org.apache.logging.log4j.layout.template.json=org.apache.logging.log4j.core --add-modules io.netty.transport.unix.common,io.netty.transport.epoll</vmOptions>
                    </configuration>
                </plugin>
                ...
            </plugins>
        </pluginManagement>
    </build>
</project>
```

If you restart the application, you should see that the HTTP server is now using the epoll transport.

```text
$ mvn inverno:run
...
2022-02-23 17:16:26,150 INFO  [main] i.i.m.h.s.i.HttpServer - HTTP Server (epoll) listening on http://0.0.0.0:8080
...
```

> The performance gain you can expect by using native transport can be significant around 10-15%.

## Step 10: Package and deploy to Docker

The application is all set, it is now time to package and deploy it to the cloud. So far, you used the Inverno Maven plugin to run the application, but it can also package the application into a native self-contained Java application including all the necessary dependencies including the Java runtime, create the corresponding Docker or CLI container image and deploy that image to a local or remote repository.

Let's create the following `install-docker` profile in the Maven project descriptor and configure the Inverno Maven plugin to build and deploy a Docker image to the local Docker repository:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>io.inverno.dist</groupId>
        <artifactId>inverno-parent</artifactId>
        <version>1.5.0-SNAPSHOT</version>
    </parent>
    <groupId>io.inverno.guide</groupId>
    <artifactId>ticket</artifactId>
    <version>1.0-SNAPSHOT</version>

    ...
    
    <profiles>
        <profile>
            <id>install-docker</id>
            <build>
                <plugins>
                    <plugin>
                        <groupId>io.inverno.tool</groupId>
                        <artifactId>inverno-maven-plugin</artifactId>
                        <executions>
                            <execution>
                                <id>build-image-docker</id>
                                <phase>install</phase>
                                <goals>
                                    <goal>build-image-docker</goal>
                                </goals>
                                <configuration>
                                    <vm>server</vm>
                                    <!-- jdk.crypto.ec: TLS, jdk.jdwp.agent: remote debug -->
                                    <addModules>jdk.crypto.ec</addModules>
                                    <executable>ticket</executable>
                                    <launchers>
                                        <launcher>
                                            <name>ticket</name>
                                            <vmOptions>-Xms2G -Xmx2G -XX:+UseNUMA -XX:+UseParallelGC --add-opens reactor.core/reactor.core.publisher=lettuce.core --add-opens org.apache.logging.log4j.core/org.apache.logging.log4j.core.jackson=com.fasterxml.jackson.databind --add-opens org.apache.logging.log4j.log4j.layout.template.json/org.apache.logging.log4j.layout.template.json=org.apache.logging.log4j.core --add-modules io.netty.transport.unix.common,io.netty.transport.epoll</vmOptions>
                                        </launcher>
                                    </launchers>
                                    <volumes>
                                        <volume>/opt/ticket/logs</volume>
                                    </volumes>
                                </configuration>
                            </execution>
                        </executions>
                    </plugin>
                </plugins>
            </build>
        </profile>
    </profiles>
</project>
```

The name of the image executable is set to `ticket` which is the name of the native executable that must be executed when running a container, it must correspond to a launcher. In the definition of the `ticket` launcher, you might have noticed that new VM options, related to memory and GC management, have been added to the ones previously defined to turn the application into a production ready application. The `jdk.crypto.ec` module has been added explicitly to have it packaged in the application's Java runtime, this module is required for TLS. Finally, volume `/opt/ticket/logs` has been defined, it corresponds to the application logs folder. This volume will be used to persist logs outside the container.

You can now install the application with the `install-docker` profile activated:

```text
$ mvn install -Pinstall-docker
...
[INFO] --- inverno-maven-plugin:${VERSION_INVERNO_TOOLS}:build-image-docker (build-image-docker) @ ticket ---
[INFO] Building project container image...
 [═══════════════════════════════════════════════ 100 % ══════════════════════════════════════════════] 
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  27.115 s
[INFO] Finished at: 2022-02-24T11:33:28+01:00
[INFO] ------------------------------------------------------------------------
```

A ticket application container image should have been created and deployed to the local Docker repository:

```text
$ docker images
REPOSITORY       TAG              IMAGE ID       CREATED         SIZE
ticket           1.0-SNAPSHOT     1837e3e29277   6 minutes ago   222MB
...
```

The corresponding native self-contained Java application generated during the build can be found in `target/maven-inverno/application_linux_amd64/ticket-1.0-SNAPSHOT` folder. You can run it just like any other application:

```text
$ ./target/maven-inverno/application_linux_amd64/ticket-1.0-SNAPSHOT/bin/ticket --profile=\"prod\"
2022-02-24 11:46:51,315 INFO  [main] i.i.g.t.App - Active profile: prod
2022-02-24 11:46:51,369 INFO  [main] i.i.c.v.Application - Inverno is starting...


     ╔════════════════════════════════════════════════════════════════════════════════════════════╗
     ║                      , ~~ ,                                                                ║
     ║                  , '   /\   ' ,                                                            ║
     ║                 , __   \/   __ ,      _                                                    ║
     ║                ,  \_\_\/\/_/_/  ,    | |  ___  _    _  ___   __  ___   ___                 ║
     ║                ,    _\_\/_/_    ,    | | / _ \\ \  / // _ \ / _|/ _ \ / _ \                ║
     ║                ,   __\_/\_\__   ,    | || | | |\ \/ /|  __/| | | | | | |_| |               ║
     ║                 , /_/ /\/\ \_\ ,     |_||_| |_| \__/  \___||_| |_| |_|\___/                ║
     ║                  ,     /\     ,                                                            ║
     ║                    ,   \/   ,                                 -- ${VERSION_INVERNO_CORE} --                  ║
     ║                      ' -- '                                                                ║
     ╠════════════════════════════════════════════════════════════════════════════════════════════╣
     ║ Java runtime        : OpenJDK Runtime Environment                                          ║
     ║ Java version        : 17+35-2724                                                           ║
     ║ Java home           : /home/jkuhn/Devel/git/winter/doc/guides/io.inverno.guide.ticket/targ ║
     ║                       et/maven-inverno/application_linux_amd64/ticket-1.0-SNAPSHOT/lib/run ║
     ║                       time                                                                 ║
     ║                                                                                            ║
     ║ Application module  : io.inverno.guide.ticket                                              ║
     ║ Application version : 1.0-SNAPSHOT                                                         ║
     ║ Application class   : io.inverno.guide.ticket.App                                          ║
     ║                                                                                            ║
     ║ Modules             :                                                                      ║
     ║  * ...                                                                                     ║
     ╚════════════════════════════════════════════════════════════════════════════════════════════╝


2022-02-24 11:46:51,373 INFO  [main] i.i.g.t.Ticket - Starting Module io.inverno.guide.ticket...
2022-02-24 11:46:51,373 INFO  [main] i.i.m.b.Boot - Starting Module io.inverno.mod.boot...
2022-02-24 11:46:51,581 INFO  [main] i.i.m.b.Boot - Module io.inverno.mod.boot started in 207ms
2022-02-24 11:46:51,581 INFO  [main] i.i.m.r.l.Lettuce - Starting Module io.inverno.mod.redis.lettuce...
2022-02-24 11:46:51,621 INFO  [main] i.i.m.r.l.Lettuce - Module io.inverno.mod.redis.lettuce started in 39ms
2022-02-24 11:46:51,621 INFO  [main] i.i.m.w.Web - Starting Module io.inverno.mod.web...
2022-02-24 11:46:51,621 INFO  [main] i.i.m.h.s.Server - Starting Module io.inverno.mod.http.server...
2022-02-24 11:46:51,622 INFO  [main] i.i.m.h.b.Base - Starting Module io.inverno.mod.http.base...
2022-02-24 11:46:51,627 INFO  [main] i.i.m.h.b.Base - Module io.inverno.mod.http.base started in 5ms
2022-02-24 11:46:52,008 INFO  [main] i.i.m.h.s.i.HttpServer - HTTP Server (epoll) listening on https://0.0.0.0:8443
2022-02-24 11:46:52,008 INFO  [main] i.i.m.h.s.Server - Module io.inverno.mod.http.server started in 387ms
2022-02-24 11:46:52,009 INFO  [main] i.i.m.w.Web - Module io.inverno.mod.web started in 387ms
2022-02-24 11:46:52,009 INFO  [main] i.i.g.t.Ticket - Module io.inverno.guide.ticket started in 637ms
2022-02-24 11:46:52,009 INFO  [main] i.i.c.v.Application - Application io.inverno.guide.ticket started in 692ms
```

## Step 11: Run the application with Docker Compose

Now that the ticket application image is in a Docker repository, it is ready to be deployed and run in the cloud on Docker, Docker Swarm or a Kubernetes cluster.

Let's create a `docker-compose.yml` file to define all services composing the application and that must be run together in an isolated environment.

```yaml
version: '3'

services:
  ticket:
    image: ticket:1.0-SNAPSHOT
    volumes:
      - logs:/opt/ticket/logs
    ports:
      - "8080:8080"
    command: --io.inverno.app.ticket.ticketAppConfiguration.redis.host=\"redis\"
  redis:
    image: redis
    volumes:
      - data:/data

volumes:
  logs:
  data:
```

The complete application is composed of the ticket application service and the Redis data store service, `logs` and `data` volumes are defined and bound to ticket application `/opt/ticket/logs` folder and Redis `/data` folder respectively.

You can now deploy the complete application using `docker-compose` command from the folder containing the `docker-compose.yml` file:

> Use `--file` and `--project-name` options if you want to run `docker-compose` from another location. By default, the project name is the name of the parent folder.

```text
$ docker-compose up -d
`Creating network "ioinvernoguideticket_default" with the default driver
Creating volume "ioinvernoguideticket_logs" with default driver
Creating volume "ioinvernoguideticket_data" with default driver
Creating ioinvernoguideticket_redis_1  ... done
Creating ioinvernoguideticket_ticket_1 ... done`
```

The `up` command initializes networks, volumes and containers, and eventually starts the applications's services. You can see that a dedicated network has been created, as well as two volumes: one to persist ticket application logs and one to persist Redis data. Data stored in volumes are not deleted when containers are stopped or removed which means application data are safe and can be easily backed up as well. Two services have been started in dedicated containers: one running the Redis data store and one running the ticket application. The 8080 port of the ticket application container is mapped to the 8080 port of the host, as a result the ticket application is accessible at [http://localhost:8080](http://localhost:8080).

You can list the two running containers and their opened ports:

```text
$ docker-compose ps
                Name                               Command               State           Ports         
-------------------------------------------------------------------------------------------------------
ioinvernoguideticket_redis_1    docker-entrypoint.sh redis ...   Up      6379/tcp              
ioinvernoguideticket_ticket_1   /opt/ticket/bin/inverno-ti ...   Up      0.0.0.0:8080->8080/tcp
```

At this stage, the application can be stopped, started or restarted using `stop`, `start` and `restart` command respectively:

```text
$ docker-compose stop
Stopping ioinvernoguideticket_redis_1  ... done
Stopping ioinvernoguideticket_ticket_1 ... done

$ docker-compose start
Starting ticket ... done
Starting redis  ... done

$ docker-compose restart
Restarting ioinvernoguideticket_redis_1  ... done
Restarting ioinvernoguideticket_ticket_1 ... done
```

If you want to undeploy the application and remove corresponding networks and containers, use the `down` command:

```text
$ docker-compose down
Stopping ioinvernoguideticket_redis_1  ... done
Stopping ioinvernoguideticket_ticket_1 ... done
Removing ioinvernoguideticket_redis_1  ... done
Removing ioinvernoguideticket_ticket_1 ... done
Removing network ioinvernoguideticket_default
```

Note that previous command didn't remove volumes, which means data are still accessible, if you reinitialize the application using the `up` command again, after an update for instance, you should see that data have been restored in the new containers.

If you wish to completely undeploy the application, you must specify the `-v` options to the `down` command to remove volumes as well:

```text
Stopping ioinvernoguideticket_redis_1  ... done
Stopping ioinvernoguideticket_ticket_1 ... done
Removing ioinvernoguideticket_redis_1  ... done
Removing ioinvernoguideticket_ticket_1 ... done
Removing network ioinvernoguideticket_default
Removing volume ioinvernoguideticket_logs
Removing volume ioinvernoguideticket_data
```

Congratulations! You've just built and deployed a Full-stack application using Inverno framework. 
