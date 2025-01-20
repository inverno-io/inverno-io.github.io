$properties(base = ../../../, title = LDAP Security Guide)

[inverno-rbac-guide]: ${base}docs/security-rbac/html/index.html
[ldap]: https://en.wikipedia.org/wiki/Lightweight_Directory_Access_Protocol
[docker]: https://www.docker.com
[docker-compose]: https://docs.docker.com/compose/
[openldap]: https://www.openldap.org/

<div class="heading">
    <h1 class="heading-title">Inverno Framework LDAP Security Guide</h1> 
    <p class="heading-subtitle">Author: <a href="mailto:jeremy.kuhn@inverno.io">Jeremy Kuhn</a></p> 
</div>

<div class="row align-items-stretch mt-5 mb-2">
    <div class="col-12 col-lg-6 mb-3">
        <div class="card shadow h-100">
            <div class="card-body p-lg-5">
                <h2 class="card-title">What you'll learn</h2>
                <p class="card-text">This guide shows how to use an LDAP authenticator to authenticate users against an <a href="https://en.wikipedia.org/wiki/Lightweight_Directory_Access_Protocol">LDAP</a> server.</p>
                <p class="card-text">It also shows how to resolve the identity of the authenticated user from the LDAP server.</p>
            </div>
        </div>
    </div>
    <div class="col-12 col-lg-6 mb-3">
        <div class="card shadow h-100">
            <div class="card-body p-lg-5">
                <h2 class="card-title">What you'll need</h2>
                <ul>
                    <li>A <em>Java™ Development Kit</em> (<a href="https://openjdk.java.net/install/">OpenJDK</a>) at least version 21.</li>
                    <li>Apache <a href="https://maven.apache.org/">Maven</a> at least version 3.9.</li>
                    <li>An <em>Integrated Development Environment</em> (IDE) such as <a href="https://www.eclipse.org/">Eclipse</a> or <a href="https://www.jetbrains.com/idea/">IDEA</a> although any text editor will do.</li>
                    <li>A secured Inverno Web application, such as the <a href="https://github.com/inverno-io/inverno-apps/tree/rbac/inverno-ticket">Inverno Ticket</a> application following the <a href="${base}docs/security-rbac/html/index.html">Inverno Framework Role-based Access Control Guide</a>.</li>
                    <li>A basic understanding of <a href="https://en.wikipedia.org/wiki/Lightweight_Directory_Access_Protocol">LDAP</a>.</li>
                </ul>
            </div>
        </div>
    </div>
</div>

$doc

This guide follows the [Inverno Framework Role-based Access Control Guide][inverno-rbac-guide]. In this guide you will replace the user authenticator backed by a Redis user repository with an LDAP authenticator to authenticate users against an [LDAP][ldap] server.

The complete Inverno Ticket application can be found in [GitHub](https://github.com/inverno-io/inverno-apps/tree/ldap/inverno-ticket).

## Step 1: Declare Inverno security LDAP dependencies

The first thing to do is to add dependencies to the Inverno LDAP module which provides an LDAP client and to the Inverno security LDAP module which provide an LDAP authenticator and an LDAP identity resolver.

These dependencies should be first added to the `pom.xml` build descriptor of the project:

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
    <version>1.0.0-SNAPSHOT</version>

    <dependencies>
        ...
        <dependency>
            <groupId>io.inverno.mod</groupId>
            <artifactId>inverno-ldap</artifactId>
        </dependency>
        <dependency>
            <groupId>io.inverno.mod</groupId>
            <artifactId>inverno-security-ldap</artifactId>
        </dependency>
        ...
    </dependencies>
</project>
```

You can now add dependencies to `io.inverno.mod.ldap` and `io.inverno.mod.security.ldap` modules in the `module-info.java` descriptor.

```java
@io.inverno.core.annotation.Module
module io.inverno.guide.ticket {
    ...
    requires io.inverno.mod.ldap;
    requires io.inverno.mod.security.ldap;
    ...
}
```

You should now be all set, and you can move on and change the authentication and identification process.

## Step 2: Authenticate with LDAP

The Inverno *security-ldap* module provides the `LDAPAuthenticator` which authenticates login credentials against an LDAP server and returns an `LDAPAuthentication` which exposes the name, DN and groups of the authenticated user.

The `LDAPAuthenticator` requires an `LDAPClient` to communicate with the LDAP server, the `LDAPClient` is provided by the *ldap* module and can then be injected in the `SecurityConfigurer` to create the authenticator. 

> The `LDAPClient` connects to `ldap://localhost:1389` by default, this can be changed by configuration.

Let's change the `SecurityConfigurer` to use the `LDAPAuthenticator` instead of the `UserAuthenticator`:

```java
package io.inverno.guide.ticket.internal.security;

import io.inverno.core.annotation.Bean;
import io.inverno.mod.base.reflect.Types;
import io.inverno.mod.base.resource.MediaTypes;
import io.inverno.mod.http.base.ExchangeContext;
import io.inverno.mod.http.base.ForbiddenException;
import io.inverno.mod.http.base.Method;
import io.inverno.mod.http.base.UnauthorizedException;
import io.inverno.mod.ldap.LDAPClient;
import io.inverno.mod.security.accesscontrol.GroupsRoleBasedAccessControllerResolver;
import io.inverno.mod.security.accesscontrol.RoleBasedAccessController;
import io.inverno.mod.security.authentication.user.UserAuthentication;
import io.inverno.mod.security.http.AccessControlInterceptor;
import io.inverno.mod.security.http.SecurityInterceptor;
import io.inverno.mod.security.http.context.InterceptingSecurityContext;
import io.inverno.mod.security.http.context.SecurityContext;
import io.inverno.mod.security.http.form.FormAuthenticationErrorInterceptor;
import io.inverno.mod.security.http.form.FormCredentialsExtractor;
import io.inverno.mod.security.http.form.FormLoginPageHandler;
import io.inverno.mod.security.http.form.RedirectLoginFailureHandler;
import io.inverno.mod.security.http.form.RedirectLoginSuccessHandler;
import io.inverno.mod.security.http.form.RedirectLogoutSuccessHandler;
import io.inverno.mod.security.http.login.LoginActionHandler;
import io.inverno.mod.security.http.login.LoginSuccessHandler;
import io.inverno.mod.security.http.login.LogoutActionHandler;
import io.inverno.mod.security.http.login.LogoutSuccessHandler;
import io.inverno.mod.security.http.token.CookieTokenCredentialsExtractor;
import io.inverno.mod.security.http.token.CookieTokenLoginSuccessHandler;
import io.inverno.mod.security.http.token.CookieTokenLogoutSuccessHandler;
import io.inverno.mod.security.identity.PersonIdentity;
import io.inverno.mod.security.identity.UserIdentityResolver;
import io.inverno.mod.security.jose.jwa.OCTAlgorithm;
import io.inverno.mod.security.jose.jws.JWSAuthentication;
import io.inverno.mod.security.jose.jws.JWSAuthenticator;
import io.inverno.mod.security.jose.jws.JWSService;
import io.inverno.mod.security.ldap.authentication.LDAPAuthentication;
import io.inverno.mod.security.ldap.authentication.LDAPAuthenticator;
import io.inverno.mod.web.server.ErrorWebRouteInterceptor;
import io.inverno.mod.web.server.WebRouteInterceptor;
import io.inverno.mod.web.server.WebRouter;
import io.inverno.mod.web.server.annotation.WebRoute;
import io.inverno.mod.web.server.annotation.WebRoutes;
import java.util.List;
import reactor.core.publisher.Mono;

@WebRoutes({
    @WebRoute(path = { "/login" }, method = { Method.GET }),
    @WebRoute(path = { "/login" }, method = { Method.POST }),
    @WebRoute(path = { "/logout" }, method = { Method.GET }, produces = { "application/json" })
})
@Bean( visibility = Bean.Visibility.PRIVATE )
public class SecurityConfigurer implements WebRouteInterceptor.Configurer<InterceptingSecurityContext<PersonIdentity, RoleBasedAccessController>>, WebRouter.Configurer<SecurityContext<PersonIdentity, RoleBasedAccessController>>, ErrorWebRouteInterceptor.Configurer<ExchangeContext> {

	private final LDAPClient ldapClient;
	private final JWSService jwsService;

	public SecurityConfigurer(LDAPClient ldapClient, JWSService jwsService) {
		this.ldapClient = ldapClient;
		this.jwsService = jwsService;
	}

	@Override
	public WebRouteInterceptor<InterceptingSecurityContext<PersonIdentity, RoleBasedAccessController>> configure(WebRouteInterceptor<InterceptingSecurityContext<PersonIdentity, RoleBasedAccessController>> interceptors) {
        ...
	}

    @Override
    public void configure(WebRoutable<SecurityContext<PersonIdentity, RoleBasedAccessController>, ?> routes) {
		routes
			.route()
                .path("/login")
                .method(Method.GET)
                .handler(new FormLoginPageHandler<>())
			.route()
                .path("/login")
                .method(Method.POST)
                .handler(new LoginActionHandler<>(
                    new FormCredentialsExtractor(),
                    new LDAPAuthenticator(this.ldapClient, "dc=inverno,dc=io")
                        .failOnDenied()
                        .flatMap(authentication -> this.jwsService.builder(LDAPAuthentication.class)
                            .header(header -> header
                                .keyId("tkt")
                                .algorithm(OCTAlgorithm.HS512.getAlgorithm())
                            )
                            .payload(authentication)
                            .build(MediaTypes.APPLICATION_JSON)
                            .map(JWSAuthentication::new)
                        ),
                    LoginSuccessHandler.of(
                        new CookieTokenLoginSuccessHandler<>(),
                        new RedirectLoginSuccessHandler<>()
                    ),
                    new RedirectLoginFailureHandler<>()
                ))
			.route()
                .path("/logout")
                .produce(MediaTypes.APPLICATION_JSON)
                .handler(new LogoutActionHandler<>(
                    authentication -> Mono.empty(),
                    LogoutSuccessHandler.of(
                        new CookieTokenLogoutSuccessHandler<>(),
                        new RedirectLogoutSuccessHandler<>()
                    )
                ));
	}

	@Override
	public ErrorWebRouteInterceptor<ExchangeContext> configure(ErrorWebRouteInterceptor<ExchangeContext> errorInterceptors) {
		...
	}
}
```

As you can see, the user repository is no longer required and the `LDAPClient` is now injected instead. The `LDAPAuthenticator` simply replaces the `UserAuthenticator` but the JWS token creation remains *almost* untouched.

> At this point you can consider completely removing the `UserRepositoryWrapper` bean as it is not used anymore.

## Step 3: Resolve identity from LDAP

The `SecurityConfigurer` is still declaring `PersonIdentity` which cannot be resolved from an `LDAPAuthentication`, as a result at this stage the application is not runnable, and you need to replace the `PersonIdentity` by the `LDAPIdentity`.

You must change this in the `SecurityConfigurer` and also replace the `UserIdentityResolver` by an `LDAPIdentityResolver` in the `SecurityInterceptor`:

```java
package io.inverno.guide.ticket.internal.security;

import io.inverno.core.annotation.Bean;
import io.inverno.mod.base.resource.MediaTypes;
import io.inverno.mod.http.base.ExchangeContext;
import io.inverno.mod.http.base.ForbiddenException;
import io.inverno.mod.http.base.Method;
import io.inverno.mod.http.base.UnauthorizedException;
import io.inverno.mod.ldap.LDAPClient;
import io.inverno.mod.security.accesscontrol.GroupsRoleBasedAccessControllerResolver;
import io.inverno.mod.security.accesscontrol.RoleBasedAccessController;
import io.inverno.mod.security.http.AccessControlInterceptor;
import io.inverno.mod.security.http.SecurityInterceptor;
import io.inverno.mod.security.http.context.InterceptingSecurityContext;
import io.inverno.mod.security.http.context.SecurityContext;
import io.inverno.mod.security.http.form.FormAuthenticationErrorInterceptor;
import io.inverno.mod.security.http.form.FormCredentialsExtractor;
import io.inverno.mod.security.http.form.FormLoginPageHandler;
import io.inverno.mod.security.http.form.RedirectLoginFailureHandler;
import io.inverno.mod.security.http.form.RedirectLoginSuccessHandler;
import io.inverno.mod.security.http.form.RedirectLogoutSuccessHandler;
import io.inverno.mod.security.http.login.LoginActionHandler;
import io.inverno.mod.security.http.login.LoginSuccessHandler;
import io.inverno.mod.security.http.login.LogoutActionHandler;
import io.inverno.mod.security.http.login.LogoutSuccessHandler;
import io.inverno.mod.security.http.token.CookieTokenCredentialsExtractor;
import io.inverno.mod.security.http.token.CookieTokenLoginSuccessHandler;
import io.inverno.mod.security.http.token.CookieTokenLogoutSuccessHandler;
import io.inverno.mod.security.jose.jwa.OCTAlgorithm;
import io.inverno.mod.security.jose.jws.JWSAuthentication;
import io.inverno.mod.security.jose.jws.JWSAuthenticator;
import io.inverno.mod.security.jose.jws.JWSService;
import io.inverno.mod.security.ldap.authentication.LDAPAuthentication;
import io.inverno.mod.security.ldap.authentication.LDAPAuthenticator;
import io.inverno.mod.security.ldap.identity.LDAPIdentity;
import io.inverno.mod.security.ldap.identity.LDAPIdentityResolver;
import io.inverno.mod.web.server.ErrorWebRouteInterceptor;
import io.inverno.mod.web.server.WebRouteInterceptor;
import io.inverno.mod.web.server.WebRouter;
import io.inverno.mod.web.server.annotation.WebRoute;
import io.inverno.mod.web.server.annotation.WebRoutes;
import java.util.List;
import reactor.core.publisher.Mono;

@WebRoutes({
    @WebRoute(path = { "/login" }, method = { Method.GET }),
    @WebRoute(path = { "/login" }, method = { Method.POST }),
    @WebRoute(path = { "/logout" }, method = { Method.GET }, produces = { "application/json" }),
})
@Bean( visibility = Bean.Visibility.PRIVATE )
public class SecurityConfigurer implements WebRouteInterceptor.Configurer<InterceptingSecurityContext<LDAPIdentity, RoleBasedAccessController>>, WebRouter.Configurer<SecurityContext<LDAPIdentity, RoleBasedAccessController>>, ErrorWebRouteInterceptor.Configurer<ExchangeContext> {

    ...
    @Override
	public WebRouteInterceptor<InterceptingSecurityContext<LDAPIdentity, RoleBasedAccessController>> configure(WebRouteInterceptor<InterceptingSecurityContext<LDAPIdentity, RoleBasedAccessController>> interceptors) {
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
                        new JWSAuthenticator<>(
                            this.jwsService,
                            LDAPAuthentication.class
                        )
                        .failOnDenied()
                        .map(jwsAuthentication -> jwsAuthentication.getJws().getPayload()),
                        new LDAPIdentityResolver(this.ldapClient),
                        new GroupsRoleBasedAccessControllerResolver()
                    ),
                    AccessControlInterceptor.authenticated()
                ))
                .intercept()
                    .path("/open-api/**")
                    .interceptor(AccessControlInterceptor.verify(securityContext -> securityContext.getAccessController()
                        .orElseThrow(() -> new ForbiddenException("Missing access controller"))
                        .hasRole("developer")
                    ));
    }

    @Override
    public void configure(WebRouter<SecurityContext<LDAPIdentity, RoleBasedAccessController>> routes) {
        ...
    }

    @Override
    public ErrorWebRouteInterceptor<ExchangeContext> configure(ErrorWebRouteInterceptor<ExchangeContext> errorInterceptors) {
        ...
    }
}
```

Note that the `GroupsRoleBasedAccessControllerResolver` can still be used with the `LDAPAuthentication` which is now wrapped in the JWS token as it implements `GroupAwareAuthentication`.

## Step 4: Run the application

You can now clean up the `TicketApp` class and remove the code related to the creation of users in the user repository as this is no longer needed.

In order to run the application, you must first set up a local LDAP server with users `jsmith` and `tktadmin` (`admin` is reserved) in `io.inverno` organization, this can be done quite easily using [Docker compose][docker-compose].

The following `docker-compose.yml` file can be used to start an [OpenLDAP][openldap] server with preset users:

```yaml
version: '3'

services:
  ldap:
    image: bitnami/openldap:2.6.2
    ports:
      - '1389:1389'
      - '1636:1636'
    environment:
      - LDAP_ROOT=dc=inverno,dc=io
    volumes:
      - ${PWD}/ldifs:/ldifs
```

File `data.ldif` file must be provided to initialize the `io.inverno` organization:

```plaintext
# inverno.io
dn: dc=inverno,dc=io
objectClass: dcObject
objectClass: organization
dc: inverno
o: inverno

# users, inverno.io
dn: ou=users,dc=inverno,dc=io
objectClass: organizationalUnit
ou: users

# jsmith, users, inverno.io
dn: cn=jsmith,ou=users,dc=inverno,dc=io
cn: jsmith
objectClass: inetOrgPerson
userPassword:: cGFzc3dvcmQ=
uid: jsmith
givenName: John
sn: Smith
displayName: John Smith
mail: jsmith@inverno.io
employeeType: dummy
title: Chief
telephoneNumber: +1 408 555 1862
mobile: +1 408 555 1941

# tktadmin, users, inverno.io
dn: cn=tktadmin,ou=users,dc=inverno,dc=io
cn: tktadmin
objectClass: inetOrgPerson
userPassword:: cGFzc3dvcmQ=
uid: tktadmin
givenName: tktadmin
sn: tktadmin
displayName: Admin
mail: tktadmin@inverno.io
employeeType: dummy

# developer, users, inverno.io
dn: cn=developer,ou=users,dc=inverno,dc=io
cn: developer
objectClass: groupOfNames
member: cn=jsmith,ou=users,dc=inverno,dc=io

# admin, users, inverno.io
dn: cn=admin,ou=users,dc=inverno,dc=io
cn: admin
objectClass: groupOfNames
member: cn=tktadmin,ou=users,dc=inverno,dc=io

```

Now create an `openldap/` directory with the following structure:

```plaintext
openldap/
├── docker-compose.yml
└── ldifs
    └── data.ldif
```

From the `openldap/` directory, you can start the OpenLDAP server with the following command:

```plaintext
$ docker-compose up -d
Creating network "openldap_default" with the default driver
Creating openldap_ldap_1 ... done
```

You can now start the Inverno Ticket application:

```plaintext
$ docker run -d -p6379:6379 redis
```

```plaintext
$ mvn inverno:run
...
[INFO] --- inverno-maven-plugin:${VERSION_INVERNO_TOOLS}:run (default-cli) @ inverno-ticket ---
 [═══════════════════════════════════════════════ 100 % ══════════════════════════════════════════════] Running project io.inverno.guide.ticket@1.0.0-SNAPSHOT...
2024-12-19 14:57:37,141 INFO  [main] i.i.g.t.TicketApp - Active profile: default
2024-12-19 14:57:37,260 INFO  [main] i.i.c.v.Application - Inverno is starting...


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
     ║ Java version        : 21.0.2+13-58                                                         ║
     ║ Java home           : /home/jkuhn/Devel/jdk/jdk-21.0.2                                     ║
     ║                                                                                            ║
     ║ Application module  : io.inverno.guide.ticket                                              ║
     ║ Application version : 1.0.0-SNAPSHOT                                                       ║
     ║ Application class   : io.inverno.guide.ticket.TicketApp                                    ║
     ║                                                                                            ║
     ║ Modules             :                                                                      ║
     ║  * ...                                                                                     ║
     ╚════════════════════════════════════════════════════════════════════════════════════════════╝


2024-12-19 14:57:37,273 INFO  [main] i.i.g.t.Ticket - Starting Module io.inverno.guide.ticket...
2024-12-19 14:57:37,274 INFO  [main] i.i.m.b.Boot - Starting Module io.inverno.mod.boot...
2024-12-19 14:57:37,582 INFO  [main] i.i.m.b.Boot - Module io.inverno.mod.boot started in 307ms
2024-12-19 14:57:37,582 INFO  [main] i.i.m.l.Ldap - Starting Module io.inverno.mod.ldap...
2024-12-19 14:57:37,589 INFO  [main] i.i.m.l.Ldap - Module io.inverno.mod.ldap started in 7ms
2024-12-19 14:57:37,590 INFO  [main] i.i.m.r.l.Lettuce - Starting Module io.inverno.mod.redis.lettuce...
2024-12-19 14:57:37,656 INFO  [main] i.i.m.r.l.Lettuce - Module io.inverno.mod.redis.lettuce started in 65ms
2024-12-19 14:57:37,656 INFO  [main] i.i.m.s.j.Jose - Starting Module io.inverno.mod.security.jose...
2024-12-19 14:57:37,754 INFO  [main] i.i.m.s.j.Jose - Module io.inverno.mod.security.jose started in 97ms
2024-12-19 14:57:37,754 INFO  [main] i.i.m.w.s.Server - Starting Module io.inverno.mod.web.server...
2024-12-19 14:57:37,754 INFO  [main] i.i.m.h.s.Server - Starting Module io.inverno.mod.http.server...
2024-12-19 14:57:37,754 INFO  [main] i.i.m.h.b.Base - Starting Module io.inverno.mod.http.base...
2024-12-19 14:57:37,759 INFO  [main] i.i.m.h.b.Base - Module io.inverno.mod.http.base started in 4ms
2024-12-19 14:57:37,760 INFO  [main] i.i.m.w.b.Base - Starting Module io.inverno.mod.web.base...
2024-12-19 14:57:37,761 INFO  [main] i.i.m.h.b.Base - Starting Module io.inverno.mod.http.base...
2024-12-19 14:57:37,761 INFO  [main] i.i.m.h.b.Base - Module io.inverno.mod.http.base started in 0ms
2024-12-19 14:57:37,763 INFO  [main] i.i.m.w.b.Base - Module io.inverno.mod.web.base started in 2ms
2024-12-19 14:57:37,809 INFO  [main] i.i.m.h.s.i.HttpServer - HTTP Server (epoll) listening on http://0.0.0.0:8080
2024-12-19 14:57:37,810 INFO  [main] i.i.m.h.s.Server - Module io.inverno.mod.http.server started in 55ms
2024-12-19 14:57:37,810 INFO  [main] i.i.m.w.s.Server - Module io.inverno.mod.web.server started in 55ms
2024-12-19 14:57:38,032 INFO  [main] i.i.g.t.Ticket - Module io.inverno.guide.ticket started in 766ms
2024-12-19 14:57:38,033 INFO  [main] i.i.c.v.Application - Application io.inverno.guide.ticket started in 888ms
```

You should be able to access the Inverno Ticker application at [http://localhost:8080](http://localhost:8080) and authenticate with users `jsmith/password` and `tktadmin/password`  which should respectively have role `developer` and `admin`:

<img src="img/inverno_ticket_authenticated_ldap.png" style="display: block; margin: 2em auto;" alt="Inverno Ticket application with LDAP"/>

If you access [http://localhost:8080/api/security/identity](http://localhost:8080/api/security/identity), you should now see an LDAP identity:

<img class="shadow" src="img/ldap_identity.png" style="display: block; margin: 2em auto;" alt="LDAP identity"/>

Congratulations! You know how to authenticate and identify users with an LDAP server in an Inverno application.
