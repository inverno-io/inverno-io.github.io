$properties(base = ../../../, title = Permission-based Access Control Guide)

[inverno-web-security-guide]: ${base}docs/security-form-jws/html/index.html
[reference-doc]: ${base}docs/release/reference/html/index.html
[redis]: https://redis.io

<div class="heading">
    <h1 class="heading-title">Inverno Framework Permission-based Access Control Guide</h1> 
    <p class="heading-subtitle">Author: <a href="mailto:jeremy.kuhn@inverno.io">Jeremy Kuhn</a></p> 
</div>

<div class="row align-items-stretch mt-5 mb-2">
    <div class="col-12 col-lg-6 mb-3">
        <div class="card shadow h-100">
            <div class="card-body p-lg-5">
                <h2 class="card-title">What you'll learn</h2>
                <p class="card-text">This guide shows how to resolve a Permission-based access controller and use it to control access to services or resources in an Inverno Web application.</p>
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
                    <li>An Inverno Web application to protect, such as the <a href="https://github.com/inverno-io/inverno-apps/tree/1.2.0/inverno-ticket">Inverno Ticket</a> application properly secured following the <a href="${base}docs/security-form-jws/html/index.html">Inverno Framework Web Spplication Security Guide</a>.</li>
                    <li>A basic understanding of Inverno's unified configuration API (see configuration module documentation in the <a href="${base}docs/release/reference/html/index/html">Reference documentation</a>)</a>.</li>
                    <li>A basic understanding of permission-based access control</a>.</li>
                </ul>
            </div>
        </div>
    </div>
</div>

$doc

This guide directly follows the [Inverno Framework Web Application Security Guide][inverno-web-security-guide]. In this guide you will continue protecting the Inverno Ticket application using permission-based access control to provide finer access control to services and resources.

The objective is to be able to control the access to particular services or resources based on the permissions that have been granted to users and stored in a [Redis][redis] data store.

The complete Inverno Ticket application can be found in [GitHub](https://github.com/inverno-io/inverno-apps/tree/pbac/inverno-ticket).

## Step 1: Resolve a Permission-based access controller

In order to control whether a user authenticated in the ticket application has access to a particular service or resource, an `AccessController` must be present in the `SecurityContext`. Following the Inverno security model, an entity accessing an application can be authenticated, maybe identified and it may be possible to control its access based on more complex approaches such as role-based or permissions-based access control. It might then not always be possible to get an access controller, this greatly depends on the authentication process and how the application is architectured.

A `PermissionBasedAccessController` allows to determine whether an authenticated entity has a particular permissions in a particular context. The fact that the current functional context is considered when evaluating permissions provides even finer access control than roles or simple permissions. The Inverno *security* module provides the `ConfigurationSourcePermissionBasedAccessController` implementation which relies on a `ConfigurationSource` to resolve such *parameterized* permissions. 

> The unified configuration API is extremely powerful and can be used in many use cases including permission-based access control since it supports parameterized configuration properties which is exactly what is needed here.

The Inverno Ticket application already uses a Redis data store for authentication, a `RedisConfigurationSource` can then be used to manage user permissions.

So the first thing to do is to resolve a `PermissionBasedAccessController` using a `ConfigurationSourcePermissionBasedAccessControllerResolver`. This must be done in the `SecurityConfigurer`, the `SecurityContext` and the `InterceptingSecurityContext` declared respectively in `WebRoutesConfigurer` and `WebInterceptorsConfigurer` should now be defined with a `PermissionBasedAccessController`, the `RedisClient` should be injected to create the underlying `RedisConfigurationSource` and a `ConfigurationSourcePermissionBasedAccessControllerResolver` should be used in the `SecurityInterceptor` to resolve the access controller:

```java
package io.inverno.app.ticket.internal.security;

...
import io.inverno.mod.configuration.source.RedisConfigurationSource;
import io.inverno.mod.redis.RedisClient;
import io.inverno.mod.security.accesscontrol.ConfigurationSourcePermissionBasedAccessControllerResolver;
import io.inverno.mod.security.accesscontrol.PermissionBasedAccessController;
...

@WebRoutes({
    @WebRoute(path = { "/login" }, method = { Method.GET }),
    @WebRoute(path = { "/login" }, method = { Method.POST }),
    @WebRoute(path = { "/logout" }, method = { Method.GET }, produces = { "application/json" }),
})
@Bean( visibility = Bean.Visibility.PRIVATE )
public class SecurityConfigurer implements WebRoutesConfigurer<SecurityContext<PersonIdentity, PermissionBasedAccessController>>, WebInterceptorsConfigurer<InterceptingSecurityContext<PersonIdentity, PermissionBasedAccessController>>, ErrorWebRouterConfigurer<ExchangeContext> {

    ...
    private final RedisConfigurationSource permissionsSource;

    public SecurityConfigurer(UserRepository<PersonIdentity, User<PersonIdentity>> userRepository, JWSService jwsService, RedisClient<String, String> redisClient) {
        this.userRepository = userRepository;
        this.jwsService = jwsService;
        this.permissionsSource = new RedisConfigurationSource(redisClient).withDefaultingStrategy(DefaultingStrategy.wildcard());
        this.permissionsSource.setKeyPrefix("SEC");
    }

    @Override
    public void configure(WebRoutable<SecurityContext<PersonIdentity, PermissionBasedAccessController>, ?> routes) {
        ...
    }

    @Override
    public void configure(WebInterceptable<InterceptingSecurityContext<PersonIdentity, PermissionBasedAccessController>, ?> interceptors) {
        interceptors
            .intercept()
                ...
                .interceptors(List.of(
                    SecurityInterceptor.of(
                        new CookieTokenCredentialsExtractor(),
                        new JWSAuthenticator<UserAuthentication<PersonIdentity>>(
                            this.jwsService,
                            Types.type(UserAuthentication.class).type(PersonIdentity.class).and().build()
                        )
                        .failOnDenied()
                        .map(jwsAuthentication -> jwsAuthentication.getJws().getPayload()),
                        new UserIdentityResolver<>(),
                        new ConfigurationSourcePermissionBasedAccessControllerResolver(this.permissionsSource)
                    ),
                    AccessControlInterceptor.authenticated()
                ));
    }
    ...
}
```

In order to facilitate the definition of permissions, the `DefaultingStrategy.wildcard()` has been applied to the `RedisConfigurationSource`. This basically allows to support defaulting when evaluating permissions, it is then possible to define default permissions that can be overridden in specific contexts, depending on how permissions are evaluated we can imagine having a user that can update all tickets except one specific ticket or the opposite.

The configuration source is also configured to use the `SEC` prefix to differentiates permissions keys from other entries (e.g. configuration entries) in the Redis data store.

> Note that it is important to define `PermissionBasedAccessController` in both `WebRoutesConfigurer` and `WebInterceptorsConfigurer`. Since the `InterceptingSecurityContext` also extends the `SecurityContext`, they cannot be defined with different bounds and the compilation will fail with inconsistent context types errors when this happens. It is also interesting to notice that it is not required to change this in the `SecurityController` since bounds could have been declared using upper wildcards (i.e. `SecurityContext<? extends PersonIdentity, ? extends AccessController>`) which is perfectly fine. Please consult the [reference documentation][reference-doc] to better understand how the exchange context is generated when defining multiple Web configurers and Web controllers.

A `PermissionBasedAccessController` should now be present in the `SecurityContext` and accessible to subsequent interceptors and route handlers.

## Step 2: Control access to services and resources

From there, you have two ways to control access to protected services and resources: you can do it globally in the `SecurityConfigurer` using a specific `AccessControlInterceptor` on specific routes or you can directly use the `PermissionBasedAccessController` in route handlers. These basically cover two different use cases: the first approach allows to control access in a non-intrusive way by configuration whereas the other one allows to implement specific behaviours based on permissions in a particular context, access control takes then an active part in the functionality.

Considering permission-based access control, the global approach might be used to evaluate permissions with no particular context or with a global context, while the second approach allows to evaluate permissions in a precise functional context. In practice it is then possible to determine whether the authenticated entity can perform a particular action like an update on a particular resource (e.g. a document) depending on some specific properties of that resource or any other contextual parameter (e.g. document type, current user's location...).

Let's start by limiting the access to the `/open-api/**` routes to users with the `access-api` permission without considering any context. You can do this globally in the `SecurityConfigurer` by defining an access control interceptor:

```java
package io.inverno.app.ticket.internal.security;

...
import io.inverno.mod.http.base.ForbiddenException;
...

@WebRoutes({
    @WebRoute(path = { "/login" }, method = { Method.GET }),
    @WebRoute(path = { "/login" }, method = { Method.POST }),
    @WebRoute(path = { "/logout" }, method = { Method.GET }, produces = { "application/json" }),
})
@Bean( visibility = Bean.Visibility.PRIVATE )
public class SecurityConfigurer implements WebRoutesConfigurer<SecurityContext<PersonIdentity, PermissionBasedAccessController>>, WebInterceptorsConfigurer<InterceptingSecurityContext<PersonIdentity, PermissionBasedAccessController>>, ErrorWebRouterConfigurer<ExchangeContext> {

    ...
    @Override
    public void configure(WebInterceptable<InterceptingSecurityContext<PersonIdentity, PermissionBasedAccessController>, ?> interceptors) {
        interceptors
            ...
            .intercept()
                .path("/open-api/**")
                .interceptor(AccessControlInterceptor.verify(securityContext -> securityContext.getAccessController()
                    .orElseThrow(() -> new ForbiddenException("Missing access controller"))
                    .hasPermission("access-api")
                ));
    }
    ...
}
```

With above configuration, only authenticated users with permission `access-api` should have access to `open-api/**` routes. Following Inverno security model, the `PermissionBasedAccessController` exposed in the `SecurityContext` is an `Optional` and `ForbiddenException` is thrown if it is empty resulting in a forbidden (403) response.

Now you can do more complex access control in the applicative code by defining a functional context when evaluating permissions. Let's say you want to control in which plan a user can push or remove a ticket, to do so you can evaluate permissions `push` or `remove` in the context defined by the targeted plan.

You can do this by injecting the `SecurityContext` in `pushTicket()` and `removeTicket()` methods in `PlanWebController`:

```java
package io.inverno.app.ticket.internal.rest.v1;

...
import io.inverno.mod.http.base.ForbiddenException;
import io.inverno.mod.security.accesscontrol.PermissionBasedAccessController;
import io.inverno.mod.security.http.context.SecurityContext;
import io.inverno.mod.security.identity.Identity;
...

@Bean( visibility = Bean.Visibility.PRIVATE )
@WebController( path = "/api/v1/plan" )
public class PlanWebController {

    ...
    @WebRoute( path = "/{planId}/ticket", method = Method.POST, consumes= MediaTypes.APPLICATION_X_WWW_FORM_URLENCODED )
    public Mono<Void> pushTicket(@PathParam long planId, @FormParam long ticketId, @FormParam Optional<Long> referenceTicketId, SecurityContext<? extends Identity, ? extends PermissionBasedAccessController> securityContext) {
        return securityContext.getAccessController()
            .orElseThrow(() -> new ForbiddenException("Missing access controller"))
            .hasPermission("push", "plan", Long.toString(planId))
                .flatMap(hasPermission -> {
                    if(!hasPermission) {
                        throw new ForbiddenException();
                    }
                    return referenceTicketId
                        .map(refTicketId -> this.planService.insertTicketBefore(planId, ticketId, refTicketId))
                        .orElse(this.planService.addTicket(planId, ticketId));
                });
    }

    @WebRoute( path = "/{planId}/ticket/{ticketId}", method = Method.DELETE, produces = MediaTypes.TEXT_PLAIN )
    public Mono<Long> removeTicket(@PathParam long planId, @PathParam long ticketId, SecurityContext<? extends Identity, ? extends PermissionBasedAccessController> securityContext) {
        return securityContext.getAccessController()
            .orElseThrow(() -> new ForbiddenException("Missing access controller"))
            .hasPermission("remove", "plan", Long.toString(planId))
            .flatMap(hasPermission -> {
                if(!hasPermission) {
                    throw new ForbiddenException();
                }
                return this.planService.removeTicket(planId, ticketId);
            });
    }
    ...
}
```

As you can see, it is quite easy to use the `SecurityContext` in pure applicative code and use it to access user's identity or control access. The expected `Identity` and `AccessController` types are also clearly specified in the Web configurers and the Web route handlers, defining a clear contract with the application. Inconsistent contexts can then easily be spotted during compilation and the application is required to provide the expected context types.

## Step 3: Manage user permissions

When using a `ConfigurationSourcePermissionBasedAccessController`, permissions are defined as configuration properties in a `ConfigurationSource`, since you are using a `RedisConfigurationSource` which is a `ConfigurableConfigurationSource`, you can then use it to set user permissions in the Redis data store.

In order to keep things simple, let's just simply add an `@Init` method to the `SecurityConfigurer` and initialize permissions for user `jsmith`:

```java
package io.inverno.app.ticket.internal.security;

...
import io.inverno.core.annotation.Init;
...

@WebRoutes({
    @WebRoute(path = { "/login" }, method = { Method.GET }),
    @WebRoute(path = { "/login" }, method = { Method.POST }),
    @WebRoute(path = { "/logout" }, method = { Method.GET }, produces = { "application/json" }),
})
@Bean( visibility = Bean.Visibility.PRIVATE )
public class SecurityConfigurer implements WebRoutesConfigurer<SecurityContext<PersonIdentity, PermissionBasedAccessController>>, WebInterceptorsConfigurer<InterceptingSecurityContext<PersonIdentity, PermissionBasedAccessController>>, ErrorWebRouterConfigurer<ExchangeContext> {

    ...
    @Init
    public void init() {
        this.permissionsSource.set("jsmith", "remove")
            .and()
            .set("jsmith", "*").withParameters("plan", "1")
            .execute()
            .blockLast();
    }
    ...
}
```

Permissions are defined for a specific user as properties with the username as name and a comma-separated list of permissions as values, `*` is used to grant all permissions. In above code, permission `remove` is granted to user `jsmith` whatever the context, this is the default set of permissions, and all permissions are granted when considering plan `1`.

> Note that permissions can also be granted to roles rather than users, a user with a given role inheriting its permissions.

Please refer to the *security* module in the [reference documentation][reference-doc] for more in-deph explanation on how permissions defaulting works and how they should be defined in a configuration source. From there, it is quite easy to imagine a user management UI targeting a `UserRepository` and a `ConfigurableConfigurationSource` exposed in a Web controller for managing both users and permissions .

> if you look into the Redis data store you should be able to see how permissions are actually defined:
> 
> ```plaintext
> redis:6379> keys SEC:PROP*
> 1) "SEC:PROP:jsmith"
> 2) "SEC:PROP:jsmith[plan=\"1\"]"
> winter-redis:6379> get "SEC:PROP:jsmith"
> "\"remove\""
> redis:6379> get "SEC:PROP:jsmith[plan=\"1\"]"
> "\"*\""
> winter-redis:6379> 
> ```

## Step 4: Run the application

you should now be able to run and test the application:

```plaintext
$ docker run -d -p6379:6379 redis
```

```plaintext
$ mvn inverno:run
...
[INFO] --- inverno-maven-plugin:${VERSION_INVERNO_TOOLS}:run (default-cli) @ inverno-ticket ---
[INFO] Running project: io.inverno.app.ticket@1.2.0-SNAPSHOT...
 [═══════════════════════════════════════════════ 100 % ══════════════════════════════════════════════] 
2022-08-12 11:57:45,750 INFO  [main] i.i.a.t.TicketApp - Active profile: default
2022-08-12 11:57:45,839 INFO  [main] i.i.c.v.Application - Inverno is starting...


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
     ║ Java version        : 17.0.2+8-86                                                          ║
     ║ Java home           : /home/jkuhn/Devel/jdk/jdk-17.0.2                                     ║
     ║                                                                                            ║
     ║ Application module  : io.inverno.app.ticket                                                ║
     ║ Application version : 1.2.0-SNAPSHOT                                                       ║
     ║ Application class   : io.inverno.app.ticket.TicketApp                                      ║
     ║                                                                                            ║
     ║ Modules             :                                                                      ║
     ║  * ...                                                                                     ║
     ╚════════════════════════════════════════════════════════════════════════════════════════════╝


2022-08-12 11:57:45,846 INFO  [main] i.i.a.t.Ticket - Starting Module io.inverno.app.ticket...
2022-08-12 11:57:45,847 INFO  [main] i.i.m.b.Boot - Starting Module io.inverno.mod.boot...
2022-08-12 11:57:46,143 INFO  [main] i.i.m.b.Boot - Module io.inverno.mod.boot started in 296ms
2022-08-12 11:57:46,143 INFO  [main] i.i.m.r.l.Lettuce - Starting Module io.inverno.mod.redis.lettuce...
2022-08-12 11:57:46,186 INFO  [main] i.i.m.r.l.Lettuce - Module io.inverno.mod.redis.lettuce started in 43ms
2022-08-12 11:57:46,187 INFO  [main] i.i.m.s.j.Jose - Starting Module io.inverno.mod.security.jose...
2022-08-12 11:57:46,254 INFO  [main] i.i.m.s.j.Jose - Module io.inverno.mod.security.jose started in 67ms
2022-08-12 11:57:46,255 INFO  [main] i.i.m.w.Web - Starting Module io.inverno.mod.web...
2022-08-12 11:57:46,255 INFO  [main] i.i.m.h.s.Server - Starting Module io.inverno.mod.http.server...
2022-08-12 11:57:46,255 INFO  [main] i.i.m.h.b.Base - Starting Module io.inverno.mod.http.base...
2022-08-12 11:57:46,260 INFO  [main] i.i.m.h.b.Base - Module io.inverno.mod.http.base started in 5ms
2022-08-12 11:57:46,906 INFO  [main] i.i.m.h.s.i.HttpServer - HTTP Server (epoll) listening on http://0.0.0.0:8080
2022-08-12 11:57:46,907 INFO  [main] i.i.m.h.s.Server - Module io.inverno.mod.http.server started in 651ms
2022-08-12 11:57:46,907 INFO  [main] i.i.m.w.Web - Module io.inverno.mod.web started in 652ms
2022-08-12 11:57:46,911 INFO  [main] i.i.a.t.Ticket - Module io.inverno.app.ticket started in 1067ms
2022-08-12 11:57:46,912 INFO  [main] i.i.c.v.Application - Application io.inverno.app.ticket started in 1159ms
```

Now if you log in as `jsmith`, you should not be able to access the Swagger UI exposing the REST API at [http://localhost:8080/open-api] since user `jsmith` does not have permission `access-api`:

<img class="shadow" src="img/inverno_ticket_openapi_forbidden.png" style="display: block; margin: 2em auto;" alt="Inverno Ticket REST API forbidden"/>

You can create two plans whose ids should be respectively `1` and `2`, you should be able to create a ticket in plan `1`:

<img class="shadow" src="img/inverno_ticket_plan1.png" style="display: block; margin: 2em auto;" alt="Inverno Ticket Plan 1"/>

But you should not be able to add this ticket to any other plan and in particular plan `2`:

<img class="shadow" src="img/inverno_ticket_plan2_forbidden.png" style="display: block; margin: 2em auto;" alt="Inverno add ticket to plan 2"/>

Congratulations! You've just used a Permission-based access controller to control access to an Inverno Web application.