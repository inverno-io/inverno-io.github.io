$properties(base = ../../../, title = Role-based Access Control Guide)

[inverno-web-security-guide]: ${base}docs/security-form-jws/html/index.html
[reference-doc]: ${base}docs/release/reference/html/index.html

<div class="heading">
    <h1 class="heading-title">Inverno Framework Role-based Access Control Guide</h1> 
    <p class="heading-subtitle">Author: <a href="mailto:jeremy.kuhn@inverno.io">Jeremy Kuhn</a></p> 
</div>

<div class="row align-items-stretch mt-5 mb-2">
    <div class="col-12 col-lg-6 mb-3">
        <div class="card shadow h-100">
            <div class="card-body p-lg-5">
                <h2 class="card-title">What you'll learn</h2>
                <p class="card-text">This guide shows how to resolve a Role-based access controller and use it to control access to services or resources in an Inverno Web application.</p>
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
                    <li>A basic understanding of <a href="https://en.wikipedia.org/wiki/Role-based_access_control">role-based access control</a>.</li>
                </ul>
            </div>
        </div>
    </div>
</div>

$doc

This guide directly follows the [Inverno Framework Web Application Security Guide][inverno-web-security-guide]. In this guide you will continue protecting the Inverno Ticket application using role-based access control to provide finer access control to services and resources.

The objective is to be able to control the access to particular services or resources based on the roles that have been assigned to users.

The complete Inverno Ticket application can be found in [GitHub](https://github.com/inverno-io/inverno-apps/tree/1.2.0/inverno-ticket).

## Step 1: Resolve a Role-based access controller

In order to control whether a user authenticated in the ticket application has access to a particular service or resource, an `AccessController` must be present in the `SecurityContext`. Following the Inverno security model, an entity accessing an application can be authenticated, maybe identified and it may be possible to control its access based on more complex approaches such as role-based or permissions-based access control. It might then not always be possible to get an access controller, this greatly depends on the authentication process and how the application is architectured.

The Inverno Ticket application uses an authentication process based on login credentials (username/password) using a `RedisUserRepository` to resolve and match the credentials provided by the user against registered users. This allows to define users and assign them into groups which can easily be used to define their roles.

So the first thing to do is to resolve a `RoleBasedAccessController` from the `UserAuthentication` resulting from the authentication. This must be done in the `SecurityConfigurer`, the `SecurityContext` and the `InterceptingSecurityContext` declared respectively in `WebRoutesConfigurer` and `WebInterceptorsConfigurer` should now be defined with a `RoleBasedAcccessController` and a `GroupsRoleBasedAccessControllerResolver` should be used in the `SecurityInterceptor` to resolve the access controller from the user's groups exposed in the `UserAuthentication`.

```java
package io.inverno.app.ticket.internal.security;

...
import io.inverno.mod.security.accesscontrol.GroupsRoleBasedAccessControllerResolver;
import io.inverno.mod.security.accesscontrol.RoleBasedAccessController;
...

@WebRoutes({
    @WebRoute(path = { "/login" }, method = { Method.GET }),
    @WebRoute(path = { "/login" }, method = { Method.POST }),
    @WebRoute(path = { "/logout" }, method = { Method.GET }, produces = { "application/json" }),
})
@Bean( visibility = Bean.Visibility.PRIVATE )
public class SecurityConfigurer implements WebRoutesConfigurer<SecurityContext<PersonIdentity, RoleBasedAccessController>>, WebInterceptorsConfigurer<InterceptingSecurityContext<PersonIdentity, RoleBasedAccessController>>, ErrorWebRouterConfigurer<ExchangeContext> {


    @Override
    public void configure(WebRoutable<SecurityContext<PersonIdentity, RoleBasedAccessController>, ?> routes) {
        ...
    }

    @Override
    public void configure(WebInterceptable<InterceptingSecurityContext<PersonIdentity, RoleBasedAccessController>, ?> interceptors) {
        interceptors
            .intercept()
                .path("/")
                .path("/api/**")
                .path("/static/**")
                .path("/webjars/**")
                .path("/open-api/**")
                .path("/logout")
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
                        new GroupsRoleBasedAccessControllerResolver()
                    ),
                    AccessControlInterceptor.authenticated()
                ));
    }
    ...
}
```

> Note that it is important to define `RoleBasedAccessController` in both `WebRoutesConfigurer` and `WebInterceptorsConfigurer`. Since the `InterceptingSecurityContext` also extends the `SecurityContext`, they cannot be defined with different bounds and the compilation will fail with inconsistent context types errors when this happens. It is also interesting to notice that it is not required to change this in the `SecurityController` since bounds could have been declared using upper wildcards (i.e. `SecurityContext<? extends PersonIdentity, ? extends AccessController>`) which is perfectly fine. Please consult the [reference documentation][reference-doc] to better understand how the exchange context is generated when defining multiple Web configurers and Web controllers.

A `RoleBasedAccessController` should now be present in the `SecurityContext` and accessible to subsequent interceptors and route handlers.

## Step 2: Control access to services and resources

From there, you have two ways to control access to protected services and resources: you can do it globally in the `SecurityConfigurer` using a specific `AccessControlInterceptor` on specific routes or you can directly use the `RoleBasedAccessController` in route handlers. These basically cover two different use cases: the first approach allows to control access in a non-intrusive way by configuration whereas the other one allows to provide different behaviours based on the user roles, access control takes then an active part in the functionality.

Let's start by limiting the access to the `/open-api/**` routes to `developer` users. You can do this globally in the `SecurityConfigurer` by defining an access control interceptor:

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
public class SecurityConfigurer implements WebRoutesConfigurer<SecurityContext<PersonIdentity, RoleBasedAccessController>>, WebInterceptorsConfigurer<InterceptingSecurityContext<PersonIdentity, RoleBasedAccessController>>, ErrorWebRouterConfigurer<ExchangeContext> {

    ...
    @Override
    public void configure(WebInterceptable<InterceptingSecurityContext<PersonIdentity, RoleBasedAccessController>, ?> interceptors) {
        interceptors
            ...
            .intercept()
                .path("/open-api/**")
                .interceptor(AccessControlInterceptor.verify(securityContext -> securityContext.getAccessController()
                    .orElseThrow(() -> new ForbiddenException("Missing access controller"))
                    .hasRole("developer")
                ));
    }
    ...
}
```

With above configuration, only authenticated users with role `developer` have access to `open-api/**` routes. Following Inverno security model, the `RoleBasedAccessController` exposed in the `SecurityContext` is an `Optional` and `ForbiddenException` is thrown if it is empty resulting in a forbidden (403) response.

You can now prevent non-admin users from creating, updating or deleting plans. Although this could be done as you have just seen, let's do it by injecting the `SecurityContext` in `createPlan()`, `updatePlan()` and `deletePlan()` methods in `PlanWebController`:

```java
package io.inverno.app.ticket.internal.rest.v1;

...
import io.inverno.mod.http.base.ForbiddenException;
import io.inverno.mod.security.accesscontrol.RoleBasedAccessController;
import io.inverno.mod.security.http.context.SecurityContext;
import io.inverno.mod.security.identity.Identity;
...

@Bean( visibility = Bean.Visibility.PRIVATE )
@WebController( path = "/api/v1/plan" )
public class PlanWebController {

    ...
    @WebRoute( method = Method.POST, consumes = MediaTypes.APPLICATION_JSON, produces = MediaTypes.APPLICATION_JSON )
    public Mono<PlanDto> createPlan(@Body PlanDto plan, WebExchange<? extends SecurityContext<? extends Identity, ? extends RoleBasedAccessController>> exchange) {
        return exchange.context().getAccessController()
            .orElseThrow(() -> new ForbiddenException("Missing access controller"))
            .hasRole("admin")
            .flatMap(hasRole -> {
                if(!hasRole) {
                    throw new ForbiddenException();
                }
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
            });
    }

    @WebRoute( path = "/{planId}", method = Method.PUT, consumes = MediaTypes.APPLICATION_JSON, produces = MediaTypes.APPLICATION_JSON )
    public Mono<PlanDto> updatePlan(@PathParam long planId, @Body PlanDto plan, SecurityContext<? extends Identity, ? extends RoleBasedAccessController> securityContext) {
        return securityContext.getAccessController()
            .orElseThrow(() -> new ForbiddenException("Missing access controller"))
            .hasRole("admin")
            .flatMap(hasRole -> {
                if (!hasRole) {
                    throw new ForbiddenException();
                }
                plan.setId(planId);
                return this.planDtoMapper.toDomain(plan)
                    .flatMap(this.planService::savePlan)
                    .flatMap(this.planDtoMapper::toDto)
                    .switchIfEmpty(Mono.error(() -> new NotFoundException()));
            });
    }

    @WebRoute( path = "/{planId}", method = Method.DELETE, produces = MediaTypes.APPLICATION_JSON )
    public Mono<PlanDto> deletePlan(@PathParam long planId, SecurityContext<? extends Identity, ? extends RoleBasedAccessController> securityContext) {
        return securityContext.getAccessController()
            .orElseThrow(() -> new ForbiddenException("Missing access controller"))
            .hasRole("admin")
            .flatMap(hasRole -> {
                if (!hasRole) {
                    throw new ForbiddenException();
                }
                return this.planService.removePlan(planId)
                    .flatMap(this.planDtoMapper::toDto)
                    .switchIfEmpty(Mono.error(() -> new NotFoundException()));
            });
    }
    ...
}
```

> Note how the `WebExchange` context type was declared in the `createPlan()` method using a wildcard, this is actually required since the actual exchange context is not strictly a `SecurityContext` (it may implements other context types).

As you can see, it is quite easy to use the `SecurityContext` in pure applicative code and use it to access user's identity or control access. The expected `Identity` and `AccessController` types are also clearly specified in the Web configurers and the Web route handlers, defining a clear contract with the application. Inconsistent contexts can then easily be spotted during compilation and the application is required to provide the expected context types.

## Step 3: Run the application

Before running the application, you should define users with roles `admin` or `developer` to test that access control is properly implemented. As described in the [Inverno Framework Web Application Security Guide][inverno-web-security-guide], you can modify `TicketApp` in order to create user `admin` in group `admin` and add user `jsmith` in the `developer` group:

```java
package io.inverno.app.ticket;

...

public class TicketApp {

    ...
    public static void main(String[] args) throws IOException {
        ...

        ticketApp.userRepository().getUser("jsmith")
            .switchIfEmpty(Mono.defer(() -> ticketApp.userRepository().createUser(User.of("jsmith")
                .password(new RawPassword("password"))
                .identity(new PersonIdentity("jsmith", "John", "Smith", "jsmith@inverno.io"))
                .groups("developer")
                .build())
            ))
            .flatMap(user -> {
                if(!user.getGroups().contains("developer")) {
                    return ticketApp.userRepository().addUserToGroups("jsmith",  "developer");
                }
                return Mono.just(user);
            })
            .block();

        ticketApp.userRepository().getUser("admin")
            .switchIfEmpty(Mono.defer(() -> ticketApp.userRepository().createUser(User.<PersonIdentity>of("admin")
                    .password(new RawPassword("password"))
                    .groups("admin")
                    .build())
            ))
            .flatMap(user -> {
                if(!user.getGroups().contains("admin")) {
                    return ticketApp.userRepository().addUserToGroups("admin",  "admin");
                }
                return Mono.just(user);
            })
            .block();
    }
}
```

Note that user `admin` was created with no identity which is perfectly acceptable and safe since the identity is exposed as an `Optional` in the `SecurityContext`.

You can now run the application:

```plaintext
$ docker run -d -p6379:6379 redis
```

```plaintext
$ mvn inverno:run
...
[INFO] --- inverno-maven-plugin:${VERSION_INVERNO_TOOLS}:run (default-cli) @ inverno-ticket ---
[INFO] Running project: io.inverno.app.ticket@1.2.0-SNAPSHOT...
 [═══════════════════════════════════════════════ 100 % ══════════════════════════════════════════════] 
2022-08-11 15:22:19,159 INFO  [main] i.i.a.t.TicketApp - Active profile: default
2022-08-11 15:22:19,238 INFO  [main] i.i.c.v.Application - Inverno is starting...


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


2022-08-11 15:22:19,242 INFO  [main] i.i.a.t.Ticket - Starting Module io.inverno.app.ticket...
2022-08-11 15:22:19,243 INFO  [main] i.i.m.b.Boot - Starting Module io.inverno.mod.boot...
2022-08-11 15:22:19,495 INFO  [main] i.i.m.b.Boot - Module io.inverno.mod.boot started in 251ms
2022-08-11 15:22:19,495 INFO  [main] i.i.m.r.l.Lettuce - Starting Module io.inverno.mod.redis.lettuce...
2022-08-11 15:22:19,539 INFO  [main] i.i.m.r.l.Lettuce - Module io.inverno.mod.redis.lettuce started in 43ms
2022-08-11 15:22:19,539 INFO  [main] i.i.m.s.j.Jose - Starting Module io.inverno.mod.security.jose...
2022-08-11 15:22:19,622 INFO  [main] i.i.m.s.j.Jose - Module io.inverno.mod.security.jose started in 82ms
2022-08-11 15:22:19,622 INFO  [main] i.i.m.w.Web - Starting Module io.inverno.mod.web...
2022-08-11 15:22:19,622 INFO  [main] i.i.m.h.s.Server - Starting Module io.inverno.mod.http.server...
2022-08-11 15:22:19,622 INFO  [main] i.i.m.h.b.Base - Starting Module io.inverno.mod.http.base...
2022-08-11 15:22:19,628 INFO  [main] i.i.m.h.b.Base - Module io.inverno.mod.http.base started in 5ms
2022-08-11 15:22:19,847 INFO  [main] i.i.m.h.s.i.HttpServer - HTTP Server (epoll) listening on http://0.0.0.0:8080
2022-08-11 15:22:19,848 INFO  [main] i.i.m.h.s.Server - Module io.inverno.mod.http.server started in 225ms
2022-08-11 15:22:19,848 INFO  [main] i.i.m.w.Web - Module io.inverno.mod.web started in 226ms
2022-08-11 15:22:19,855 INFO  [main] i.i.a.t.Ticket - Module io.inverno.app.ticket started in 614ms
2022-08-11 15:22:19,857 INFO  [main] i.i.c.v.Application - Application io.inverno.app.ticket started in 694ms
```

Now if you log in as `jsmith`, you should be able to access the Swagger UI exposing the REST API at [http://localhost:8080/open-api]:

<img class="shadow" src="img/inverno_ticket_openapi.png" style="display: block; margin: 2em auto;" alt="Inverno Ticket REST API"/>

But you should not be able to create plans:

<img class="shadow" src="img/inverno_ticket_plan_forbidden.png" style="display: block; margin: 2em auto;" alt="Inverno Ticket Create plan forbidden"/>

If you logout and log in as `admin`, you should be able to create plans but no longer to access the Swagger UI:

<img class="shadow" src="img/inverno_ticket_openapi_forbidden.png" style="display: block; margin: 2em auto;" alt="Inverno Ticket REST API forbidden"/>

Congratulations! You've just used a Role-based access controller to control access to an Inverno Web application.