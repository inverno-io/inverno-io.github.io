$properties(base = ../../../, title = Web Application Security Guide)

[inverno-full-stack-guide]: ${base}docs/redis-vue3-fullstack/html/index.html
[open-apis]: https://www.openapis.org
[webjars]: https://www.webjars.org
[rfc7515]: https://datatracker.ietf.org/doc/html/rfc7515
[rfc7519]: https://datatracker.ietf.org/doc/html/rfc7519
[redis]: https://redis.io
[pbkdf2]: https://en.wikipedia.org/wiki/PBKDF2
[docker]: https://www.docker.com
[docker-compose]: https://docs.docker.com/compose/

<div class="heading">
    <h1 class="heading-title">Inverno Framework Web Application Security Guide</h1> 
    <p class="heading-subtitle">Author: <a href="mailto:jeremy.kuhn@inverno.io">Jeremy Kuhn</a></p> 
</div>

<div class="row align-items-stretch mt-5 mb-2">
    <div class="col-12 col-lg-6 mb-3">
        <div class="card shadow h-100">
            <div class="card-body p-lg-5">
                <h2 class="card-title">What you'll learn</h2>
                <p class="card-text">This guide shows how to secure access to an Inverno Web application.</p>
                <p class="card-text">We will guide you through setting up a form-based login in an Inverno Web application using a <a href="https://redis.io">Redis</a> user repository.</p>
                <p class="card-text">On successful login, a <a href="https://datatracker.ietf.org/doc/html/rfc7515">JSON Web Signature</a> token will be generated and returned to the Web browser in a cookie, this token will then be used to access protected services and resources.</p>
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
                    <li>An Inverno Web application to secure, such as the <a href="https://github.com/inverno-io/inverno-apps/tree/master/inverno-ticket">Inverno Ticket</a> application you should have created when you followed the <a href="${base}docs/redis-vue3-fullstack/html/index.html">Inverno Framework Full-Stack application Guide</a>.</li>
                    <li>A basic understanding of Redis data store.</li>
                    <li>A basic understanding of Inverno's Web application development (see web module documentation in the <a href="${base}docs/release/reference/html/index/html">Reference documentation</a>).</li>
                    <li>A basic understanding of JSON Object Signing and Encryption concepts.</li>
                </ul>
            </div>
        </div>
    </div>
</div>

$doc

This guide directly follows the [Inverno Framework Full-Stack application Guide][inverno-full-stack-guide]. In this guide you will secure the Inverno Ticket application that was created then.

The objective is to protect the access to every service or resources exposed in the application by requiring an authentication. Since you clearly don't want to ask a user to manually enter credentials each time a service or a resource is accessed, authentication will be done in two steps: first a user will have to authenticate on a login page by providing proper credentials (username/password) submitted in a form to a login action which then returns a token to the client (Web browser) sent in every subsequent requests to the application.

The login action will authenticate login credentials against a [Redis][redis] user repository. A redis data store should already be configured in the Ticket application making things easier.

The token will be a [JSON Web Signature][rfc7515] object that can be easily authenticated by validating a digital signature. This token can be seen as a visitor badge that one would have obtained form a security desk at the entrance of building after showing proper credentials (e.g. ID card...). 

The complete Inverno Ticket application can be found in [GitHub](https://github.com/inverno-io/inverno-apps/tree/secure/inverno-ticket).

## Step 1: Declare Inverno security dependencies

The first thing to do is to add dependencies to Inverno security modules. In order to secure a Web application using form-based login and a JWS token you basically need to add dependencies to Inverno *security-http* module, which provides services to secure Web applications, and to Inverno *securiy-jose* module, which provides services for creating and validating JWS, JWE or JWT tokens.

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
            <artifactId>inverno-security-http</artifactId>
        </dependency>
        <dependency>
            <groupId>io.inverno.mod</groupId>
            <artifactId>inverno-security-jose</artifactId>
        </dependency>
        ...
    </dependencies>
</project>
```

You can now add dependencies to `io.inverno.mod.security.http` and `io.inverno.mod.security.jose` modules in the `module-info.java` descriptor.

```java
@io.inverno.core.annotation.Module
module io.inverno.guide.ticket {
    requires io.inverno.mod.boot;
    requires io.inverno.mod.redis.lettuce;
    requires io.inverno.mod.web.server;
    requires io.inverno.mod.security.http;
    requires io.inverno.mod.security.jose;

    requires org.apache.logging.log4j;
    requires org.apache.logging.log4j.layout.template.json;

    exports io.inverno.guide.ticket.internal.model to com.fasterxml.jackson.databind;
    exports io.inverno.guide.ticket.internal.rest.v1.dto to com.fasterxml.jackson.databind;
}

```

You should now be all set, and you can start securing the application.

> Note that the Inverno *security* module, which defines core security API and services, is provided as a transitive dependency.

## Step 2: Set up the Login page

Let's start by setting up the application login page that will be used by users to log into the application. The Inverno *security-http* module provides a whitelabel login page through the `FormLoginPageHandler`, it contains a simple login form which allows users to submit login credentials (username/password) to the login action.

The login action is implemented in the `LoginActionHandler` which basically authenticates the submitted credentials using a `FormCredentialsExtractor` to extract `LoginCredentials` from the request and an `Authenticator` capable of authenticating `LoginCredentials`. Users credentials will be authenticated against a Redis user repository, as a result a `UserAuthenticator` backed by a `RedisUserRepository` shall be used. Since the `User` stored in the repository can also provide identity information, `PersonIdentity` information will also be stored and exposed in order to be used later in the application whenever there is a need for identification.

On successful authentication, a [JSON Web Signature][rfc7515] object wrapping the resulting `UserAuthentication<PersonIdentity>` should be created and returned in a cookie to the Web browser.

You can start by creating bean `UserRepositoryWrapper` to expose the `RedisUserRepository`. This will abstract the actual user repository from the user authenticator and ease maintenance if the `UserRepository` implementation must be changed at some point. A `RedisUserRepository` requires a `RedisClient` and an `ObjectMapper` which should already be provided in the application by the *redis* and the *boot* modules respectively.

```java
package io.inverno.guide.ticket.internal.security;

import com.fasterxml.jackson.databind.ObjectMapper;
import io.inverno.core.annotation.Bean;
import io.inverno.core.annotation.Wrapper;
import io.inverno.mod.redis.RedisClient;
import io.inverno.mod.security.authentication.user.RedisUserRepository;
import io.inverno.mod.security.authentication.user.User;
import io.inverno.mod.security.authentication.user.UserRepository;
import io.inverno.mod.security.identity.PersonIdentity;
import java.util.function.Supplier;

/**
 * <p>
 * The user repository managing application users and used to authenticate users.
 * </p>
 *
 * @author <a href="mailto:jeremy.kuhn@inverno.io">Jeremy Kuhn</a>
 */
@Wrapper @Bean( name = "userRepository" )
public class UserRepositoryWrapper implements Supplier<UserRepository<PersonIdentity, User<PersonIdentity>>> {

    private final RedisClient<String, String> redisClient;
    private final ObjectMapper mapper;

    public UserRepositoryWrapper(RedisClient<String, String> redisClient, ObjectMapper mapper) {
        this.redisClient = redisClient;
        this.mapper = mapper;
    }

    @Override
    public UserRepository<PersonIdentity, User<PersonIdentity>> get() {
        return new RedisUserRepository<>(this.redisClient, this.mapper);
    }
}

```

Now you need to define a JSON Web Key (JWK) for your application, this key will be used to create, parse and verify the JSON Web Signature tokens. The Inverno *security-jose* module provides the `JWKService` which can be used to generate a `JWK` or load it from the configuration. To keep things simple, let's assume a new key will be generated each time the application is started.

The algorithm used to digitally sign JWS will be a simple symmetric algorithm, such as `HS512`. The `JWSService` that will be used to manipulate JWS objects has the ability to automatically look up keys by id from a `JWKStore`. So let's create an `InMemoryJWKStore` bean to store keys in memory and a trusted `JWK` bean registered into the `JWKStore`.

```java
package io.inverno.guide.ticket.internal.security;

import io.inverno.core.annotation.Bean;
import io.inverno.core.annotation.Wrapper;
import io.inverno.mod.security.jose.jwa.OCTAlgorithm;
import io.inverno.mod.security.jose.jwk.InMemoryJWKStore;
import io.inverno.mod.security.jose.jwk.JWK;
import io.inverno.mod.security.jose.jwk.JWKService;
import io.inverno.mod.security.jose.jwk.JWKStore;
import java.util.function.Supplier;

@Wrapper @Bean( name = "jwk" )
public class JWKWrapper implements Supplier<JWK> {

	private final JWKService jwkService;

	public JWKWrapper(JWKService jwkService) {
		this.jwkService = jwkService;
	}

	@Override
	public JWK get() {
		return this.jwkService.oct().generator()
			.keyId("tkt")
			.algorithm(OCTAlgorithm.HS512.getAlgorithm())
			.generate()
			.map(JWK::trust)
			.flatMap(jwk -> jwkService.store().set(jwk).thenReturn(jwk))
			.block();
	}

	@Wrapper @Bean
	public static class JWKStoreWrapper implements Supplier<JWKStore> {

		@Override
		public JWKStore get() {
			return new InMemoryJWKStore();
		}
	}
}

```

Note that the generated key must be explicitly trusted because a generated keys are not trusted by default. Untrusted keys cannot be used to create or verify JWS objects.

> You may wonder why we had to create two beans and why not just the `JWKStore` bean with a `JWK` socket or an init method for initializing the JWK. The reason is actually quite simple, since the `JWKStore` is injected in the `JWKService`, it is not possible to create or inject the `JWK` in the `JWKStore` using the `JWKService` as this would lead to a cycle in the bean dependency graph.

From there, it is quite simple to load a JWK from configuration, it only requires to use a `builder()` instead of a `generator()` and provide the symmetric key value in the configuration.

You can now create the `SecurityConfigurer` bean that will be used to configure security for the whole application. For now, it shall implement `WebRouter.Configurer<SecurityContext<PersonIdentity, AccessController>>` in order to define routes to the login page using the `FormLoginPageHandler` and the login action using the `LoginActionHandler`:

```java
package io.inverno.guide.ticket.internal.security;

import io.inverno.core.annotation.Bean;
import io.inverno.mod.base.resource.MediaTypes;
import io.inverno.mod.http.base.Method;
import io.inverno.mod.security.accesscontrol.AccessController;
import io.inverno.mod.security.authentication.LoginCredentialsMatcher;
import io.inverno.mod.security.authentication.user.User;
import io.inverno.mod.security.authentication.user.UserAuthentication;
import io.inverno.mod.security.authentication.user.UserAuthenticator;
import io.inverno.mod.security.authentication.user.UserRepository;
import io.inverno.mod.security.http.context.SecurityContext;
import io.inverno.mod.security.http.form.FormCredentialsExtractor;
import io.inverno.mod.security.http.form.FormLoginPageHandler;
import io.inverno.mod.security.http.form.RedirectLoginFailureHandler;
import io.inverno.mod.security.http.form.RedirectLoginSuccessHandler;
import io.inverno.mod.security.http.login.LoginActionHandler;
import io.inverno.mod.security.http.login.LoginSuccessHandler;
import io.inverno.mod.security.http.token.CookieTokenLoginSuccessHandler;
import io.inverno.mod.security.identity.PersonIdentity;
import io.inverno.mod.security.jose.jwa.OCTAlgorithm;
import io.inverno.mod.security.jose.jws.JWSAuthentication;
import io.inverno.mod.security.jose.jws.JWSService;
import io.inverno.mod.web.server.WebRouter;
import io.inverno.mod.web.server.annotation.WebRoute;
import io.inverno.mod.web.server.annotation.WebRoutes;

@WebRoutes({
    @WebRoute(path = { "/login" }, method = { Method.GET }),
    @WebRoute(path = { "/login" }, method = { Method.POST }),
})
@Bean( visibility = Bean.Visibility.PRIVATE )
public class SecurityConfigurer implements WebRouter.Configurer<SecurityContext<PersonIdentity, AccessController>> {

    private final UserRepository<PersonIdentity, User<PersonIdentity>> userRepository;
    private final JWSService jwsService;

    public SecurityConfigurer(UserRepository<PersonIdentity, User<PersonIdentity>> userRepository, JWSService jwsService) {
        this.userRepository = userRepository;
        this.jwsService = jwsService;
    }

    @Override
	public void configure(WebRouter<SecurityContext<PersonIdentity, AccessController>> routes) {
        routes
            .route()                                                                                                                     // 1
                .path("/login")
                .method(Method.GET)
                .handler(new FormLoginPageHandler<>())
            .route()                                                                                                                     // 2
                .path("/login")
                .method(Method.POST)
                .handler(new LoginActionHandler<>(                                                                                       // 3
                    new FormCredentialsExtractor(),                                                                                      // 4
                    new UserAuthenticator<>(this.userRepository, new LoginCredentialsMatcher<>())                                        // 5
                        .failOnDenied()                                                                                                  // 6
                        .flatMap(authentication -> this.jwsService.<UserAuthentication<PersonIdentity>>builder(UserAuthentication.class) // 7
                            .header(header -> header
                                .keyId("tkt")
                                .algorithm(OCTAlgorithm.HS512.getAlgorithm())
                            )
                            .payload(authentication)
                            .build(MediaTypes.APPLICATION_JSON)
                            .map(JWSAuthentication::new)
                        ),
                    LoginSuccessHandler.of(                                                                                              // 8
                        new CookieTokenLoginSuccessHandler<>(),
                        new RedirectLoginSuccessHandler<>()
                    ),
                    new RedirectLoginFailureHandler<>()                                                                                  // 9
                ));
    }
}
```

Above configurer deserves some deeper explanations:

1. The `GET /login` web route is declared to serve the whitelabel login page using the `FormLoginPageHandler`.
2. The `POST /login` web route is declared to process the login action, it is basically targeted by the form in the login page.
3. The login action is handled by the `LoginActionHandler`.
4. A `FormCredentialsExtractor` is used to extract the `LoginCredentials` from the request.
5. A `UserAuthenticator` using the user repository is used to authenticate the submitted credentials using a `LoginCredentialsMatcher` to match credentials.
6. The authenticator authenticates the credentials and returns a resulting `Authentication` which can be either denied (failed authentication), anonymous (no authentication) or authenticated (successful authentication). The `failOnDenied()` operator basically fails the process in case of denied authentication and no further processing is then performed, resulting in an `UnauthorizedException` being thrown.
7. If successful, the resulting `UserAuthentication` is mapped into a `JWSAuthentication` which basically holds a `JWS` with the authentication as payload and created using the `tkt` key previously stored in the `JWKStore`.
8. On successful login, a `CookieTokenLoginSuccessHandler` is used to return an `AUTH-TOKEN` cookie to the Web browser containing the JWS compact representation. The `RedirectLoginSuccessHandler` is also used to redirect the user to its original request or to `/` if the login page was directly accessed.
9. Finally, a `RedirectLoginFailureHandler` is used to redirect the user to the login page in case of denied authentication.

You should also notice that the exchange context type declared in `WebRouter.Configurer` is `SecurityContext<PersonIdentity, AccessController>`. Although access to the `SecurityContext` is not required by the `FormLoginPageHandler` or the `LoginActionHandler`, it is required if you want to expose a logout action using the `LogoutActionHandler` in order to be able to release the security context.

Let's define a `/logout` route using the `LogoutActionHandler` to log out users:

```java
package io.inverno.guide.ticket.internal.security;

import io.inverno.core.annotation.Bean;
import io.inverno.mod.base.resource.MediaTypes;
import io.inverno.mod.http.base.Method;
import io.inverno.mod.security.accesscontrol.AccessController;
import io.inverno.mod.security.authentication.LoginCredentialsMatcher;
import io.inverno.mod.security.authentication.user.User;
import io.inverno.mod.security.authentication.user.UserAuthentication;
import io.inverno.mod.security.authentication.user.UserAuthenticator;
import io.inverno.mod.security.authentication.user.UserRepository;
import io.inverno.mod.security.http.context.SecurityContext;
import io.inverno.mod.security.http.form.FormCredentialsExtractor;
import io.inverno.mod.security.http.form.FormLoginPageHandler;
import io.inverno.mod.security.http.form.RedirectLoginFailureHandler;
import io.inverno.mod.security.http.form.RedirectLoginSuccessHandler;
import io.inverno.mod.security.http.form.RedirectLogoutSuccessHandler;
import io.inverno.mod.security.http.login.LoginActionHandler;
import io.inverno.mod.security.http.login.LoginSuccessHandler;
import io.inverno.mod.security.http.login.LogoutActionHandler;
import io.inverno.mod.security.http.login.LogoutSuccessHandler;
import io.inverno.mod.security.http.token.CookieTokenLoginSuccessHandler;
import io.inverno.mod.security.http.token.CookieTokenLogoutSuccessHandler;
import io.inverno.mod.security.identity.PersonIdentity;
import io.inverno.mod.security.jose.jwa.OCTAlgorithm;
import io.inverno.mod.security.jose.jws.JWSAuthentication;
import io.inverno.mod.security.jose.jws.JWSService;
import io.inverno.mod.web.server.WebRouter;
import io.inverno.mod.web.server.annotation.WebRoute;
import io.inverno.mod.web.server.annotation.WebRoutes;
import reactor.core.publisher.Mono;

@WebRoutes({
    @WebRoute(path = { "/login" }, method = { Method.GET }),
    @WebRoute(path = { "/login" }, method = { Method.POST }),
    @WebRoute(path = { "/logout" }, method = { Method.GET }, produces = { "application/json" })
})
@Bean( visibility = Bean.Visibility.PRIVATE )
public class SecurityConfigurer implements WebRouter.Configurer<SecurityContext<PersonIdentity, RoleBasedAccessController>> {

    private final UserRepository<PersonIdentity, User<PersonIdentity>> userRepository;
    private final JWSService jwsService;

    public SecurityConfigurer(UserRepository<PersonIdentity, User<PersonIdentity>> userRepository, JWSService jwsService) {
        this.userRepository = userRepository;
        this.jwsService = jwsService;
    }

    @Override
    public void configure(WebRouter<SecurityContext<PersonIdentity, AccessController>> routes) {
        routes
            ...
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
}
```

The above logout action basically does nothing to release the security context since the application stores its state in a JWS and is basically stateless. On successful logout, which should then always be the case, the cookie is removed by the `CookieTokenLogoutSuccessHandler` and the Web browser redirected to `/` using the `RedirectLogoutSuccessHandler`.

The login page should now be all set with login and logout actions, and you can move on by securing access to services and resources.

## Step 3: Secure applications services and resources

You should now identify the services or resources in the application that should be protected and require an authentication to access.

The Ticket application basically exposes the following routes:

- `/` which points to the application welcome page.
- `/api/**` which points to the REST services exposed by the `PlanWebController` and the `TicketWebController`.
- `/static/**` which points to the static resources under `web_root` resource.
- `/webjars/**` which points to [WebJars][webjars] resources.
- `/open-api/**` which points to the [OpenAPI][open-apis] specifications.
- `/login` which points to the login page.
- `/logout` which points to the logout action

All these routes except the `/login` page, which must remain accessible to any user, should be protected against unauthenticated or anonymous access. In order to do that, you need to intercept requests to these routes and reject those that do not contain valid credentials, namely the `AUTH-TOKEN` cookie containing a compact JWS normally created in the login action. In case valid credentials have been provided, a `SecurityContext` should be created and injected in the exchange context, the security context is then accessible to subsequent interceptors and route handlers.

You must then modify the `SecurityConfigurer` and make it implement `WebRouteInterceptor.Configurer<InterceptingSecurityContext<PersonIdentity, AccessController>>` in order to configure the security interceptors on the protected routes.

```java
package io.inverno.guide.ticket.internal.security;

import io.inverno.core.annotation.Bean;
import io.inverno.mod.base.reflect.Types;
import io.inverno.mod.base.resource.MediaTypes;
import io.inverno.mod.http.base.Method;
import io.inverno.mod.security.accesscontrol.AccessController;
import io.inverno.mod.security.authentication.LoginCredentialsMatcher;
import io.inverno.mod.security.authentication.user.User;
import io.inverno.mod.security.authentication.user.UserAuthentication;
import io.inverno.mod.security.authentication.user.UserAuthenticator;
import io.inverno.mod.security.authentication.user.UserRepository;
import io.inverno.mod.security.http.AccessControlInterceptor;
import io.inverno.mod.security.http.SecurityInterceptor;
import io.inverno.mod.security.http.context.InterceptingSecurityContext;
import io.inverno.mod.security.http.context.SecurityContext;
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
import io.inverno.mod.security.jose.jwa.OCTAlgorithm;
import io.inverno.mod.security.jose.jws.JWSAuthentication;
import io.inverno.mod.security.jose.jws.JWSAuthenticator;
import io.inverno.mod.security.jose.jws.JWSService;
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
public class SecurityConfigurer implements WebRouteInterceptor.Configurer<InterceptingSecurityContext<PersonIdentity, AccessController>>, WebRouter.Configurer<SecurityContext<PersonIdentity, AccessController>> {

    ...
    @Override
    public WebRouteInterceptor<InterceptingSecurityContext<PersonIdentity, AccessController>> configure(WebRouteInterceptor<InterceptingSecurityContext<PersonIdentity, AccessController>> interceptors) {
        return interceptors
            .intercept()                                                                                  // 1
                .path("/")
                .path("/api/**")
                .path("/static/**")
                .path("/webjars/**")
                .path("/open-api/**")
                .path("/logout")
                .interceptors(List.of(
                    SecurityInterceptor.of(                                                               // 2
                        new CookieTokenCredentialsExtractor(),                                            // 3
                        new JWSAuthenticator<UserAuthentication<PersonIdentity>>(                         // 4
                            this.jwsService,
                            Types.type(UserAuthentication.class).type(PersonIdentity.class).and().build()
                        )
                        .failOnDenied()                                                                   // 5
                        .map(jwsAuthentication -> jwsAuthentication.getJws().getPayload())                // 6
                    ),
                    AccessControlInterceptor.authenticated()                                              // 7
                ));
    }
}
```

1. All routes except the `/login` route are intercepted.
2. The `SecurityInterceptor` which is the main security interceptor responsible for authenticating credentials and creating the security context is declared first.
3. It uses a `CookieTokenCredentialsExtractor` to extract `TokenCredentials` from the `AUTH-TOKEN` cookie in the request.
4. A `JWSAuthenticator` backed by the `JWSService` is then used to parse and verify the compact JWS token which should wrap the original `UserAuthentication<PersonIdentity>` that was created when authenticating the user credentials in the login action. Note that the type of the JWS payload must be specified explicitly for the underlying `ObjectMapper` to decode it (and also to benefit from static checking).
5. The `failOnDenied()` operator is used to stop the processing if the resulting `JWSAuthentication` was denied either because the token was invalid (e.g. not a JWS or bad signature) or because the wrapped authentication is actually denied (which should never happen since a token is only generated on successful login credentials authentication), resulting in an `UnauthorizedException` being thrown.
6. The original `UserAuthentication<PersonIdentity>` is then unwrapped, the security interceptor then creates a `SecurityContext` with the original authentication instead of the `JWSAuthentication`.
7. Finally, we make sure only authenticated security context can access the protected service or resource. This basically means that only users that provided valid login credentials have access to the application.

Note that the exchange context type declared in the `WebRouteInterceptor.Configurer` is `InterceptingSecurityContext<PersonIdentity, AccessController>`, the `InterceptingSecurityContext` is required by the `SecurityInterceptor` in order to populate the `SecurityContext`.

Using a `JWS` token has many advantages: first it is easy to verify that the provided credentials are valid by verifying the digital signature using `tkt` key and then, the original `UserAuthentication<PersonIdentity>` could have been restored. It provides user's identity and roles that can then be used directly to get the user's `PersonIdentity` or obtain a `RoleBasedAccessController` without having to query the user repository again.

> Keep in mind that this is a basic use case that might not be secured enough as JWS tokens thus created never expire which should be seen as a security issue in most systems. It should then be wise to consider using a [JSON Web Token][rfc7519] which supports expiry or implement such expiration mechanism explicitly.

It should now be impossible to access a protected resource without being authenticated, however a user won't be automatically redirected to the login page when receiving an unauthorized (401) error. This can be done by intercepting `UnauthorizedException` in the error router.

You must then modify the `SecurityConfigurer` one more time to make it implement `ErrorWebRouteInterceptor.Configurer<ExchangeContext>` in order to intercept unauthorized errors and use a `FormAuthenticationErrorInterceptor` to redirect the user to the login page:

```java
package io.inverno.guide.ticket.internal.security;

import io.inverno.core.annotation.Bean;
import io.inverno.mod.base.reflect.Types;
import io.inverno.mod.base.resource.MediaTypes;
import io.inverno.mod.http.base.ExchangeContext;
import io.inverno.mod.http.base.Method;
import io.inverno.mod.http.base.UnauthorizedException;
import io.inverno.mod.security.accesscontrol.AccessController;
import io.inverno.mod.security.authentication.LoginCredentialsMatcher;
import io.inverno.mod.security.authentication.user.User;
import io.inverno.mod.security.authentication.user.UserAuthentication;
import io.inverno.mod.security.authentication.user.UserAuthenticator;
import io.inverno.mod.security.authentication.user.UserRepository;
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
import io.inverno.mod.security.jose.jwa.OCTAlgorithm;
import io.inverno.mod.security.jose.jws.JWSAuthentication;
import io.inverno.mod.security.jose.jws.JWSAuthenticator;
import io.inverno.mod.security.jose.jws.JWSService;
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
public class SecurityConfigurer implements WebRouteInterceptor.Configurer<InterceptingSecurityContext<PersonIdentity, AccessController>>, WebRouter.Configurer<SecurityContext<PersonIdentity, AccessController>>, ErrorWebRouteInterceptor.Configurer<ExchangeContext> {

    ...
	@Override
	public ErrorWebRouteInterceptor<ExchangeContext> configure(ErrorWebRouteInterceptor<ExchangeContext> errorInterceptors) {
		return errorInterceptors
			.interceptError()
                .path("/")
                .error(UnauthorizedException.class)
                .interceptor(new FormAuthenticationErrorInterceptor<>());
	}
}
```

The Inverno Ticket application is a single page application, so this redirection can be limited to the welcome page `/`, accessing any other resources will simply result in an authorized (401) response.

The Ticket application should now be properly secured and only authenticated users shall be allowed to access the application. 

## Step 4: Expose the user's identity

You might now want to display the user's identity in the application UI. To do so you need to expose the identity on the server. 

But first you must resolve the user's identity from the `UserAuthentication`, this can be done easily by adding a `UserIdentityResolver` to the `SecurityInterceptor`:

```java
package io.inverno.guide.ticket.internal.security;

import io.inverno.core.annotation.Bean;
import io.inverno.mod.base.reflect.Types;
import io.inverno.mod.base.resource.MediaTypes;
import io.inverno.mod.http.base.ExchangeContext;
import io.inverno.mod.http.base.Method;
import io.inverno.mod.http.base.UnauthorizedException;
import io.inverno.mod.security.accesscontrol.AccessController;
import io.inverno.mod.security.authentication.LoginCredentialsMatcher;
import io.inverno.mod.security.authentication.user.User;
import io.inverno.mod.security.authentication.user.UserAuthentication;
import io.inverno.mod.security.authentication.user.UserAuthenticator;
import io.inverno.mod.security.authentication.user.UserRepository;
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
public class SecurityConfigurer implements WebRoutesConfigurer<SecurityContext<PersonIdentity, AccessController>>, WebInterceptorsConfigurer<InterceptingSecurityContext<PersonIdentity, AccessController>>, ErrorWebRouterConfigurer<ExchangeContext> {

    ...
    @Override
    public void configure(WebInterceptable<InterceptingSecurityContext<PersonIdentity, AccessController>, ?> interceptors) {
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
                        new UserIdentityResolver<>()
                    ),
                    AccessControlInterceptor.authenticated()
                ));
    }
}
```

> Here the user's identity is resolved for all routes, but it is also possible to resolve it only for the routes that need it, simply by defining different security interceptors.

The user's identity should now be accessible from the `SecurityContext` in the exchange, you can then create the following `SecurityController` to expose the user's identity to the UI:

```java
package io.inverno.guide.ticket.internal.security;

import io.inverno.core.annotation.Bean;
import io.inverno.mod.base.resource.MediaTypes;
import io.inverno.mod.http.base.Method;
import io.inverno.mod.http.base.NotFoundException;
import io.inverno.mod.security.accesscontrol.AccessController;
import io.inverno.mod.security.http.context.SecurityContext;
import io.inverno.mod.security.identity.Identity;
import io.inverno.mod.web.server.annotation.WebController;
import io.inverno.mod.web.server.annotation.WebRoute;

@Bean( visibility = Bean.Visibility.PRIVATE )
@WebController( path = "/api/security" )
public class SecurityController {

    @WebRoute( path = "/identity", method = Method.GET, produces = MediaTypes.APPLICATION_JSON )
    public Identity identity(SecurityContext<? extends Identity, ? extends AccessController> securityContext) {
        return securityContext.getIdentity().orElseThrow(() -> new NotFoundException("Identify not found"));
    }
}
```

User's identity is now exposed at [http://localhost:8080/api/security/identity](http://localhost:8080/api/security/identity). The UI can also be modified to display the user's identity.

> One might say that the JWS token actually contains user's identity information and that there is then no need to expose them on the server, however an `AUTH-TOKEN` cookie must be HTTP only and therefore cannot be read from the UI.

## Step 5: Run the application

Before you can run the application, you need to define a user in the Redis user repository. Although it is possible to do this by creating an entry in the Redis data store using a simple Redis client, it is easier and safer to rely on the `UserRepository` to do this.

So let's modify the `TicketApp#main()` method and use the `UserRepository` bean to create user `jsmith` if it does not already exist when the application is starting:

```java
package io.inverno.guide.ticket;

import io.inverno.core.annotation.Bean;
import io.inverno.core.v1.Application;
import io.inverno.mod.configuration.ConfigurationKey;
import io.inverno.mod.configuration.ConfigurationSource;
import io.inverno.mod.configuration.source.BootstrapConfigurationSource;
import io.inverno.mod.security.authentication.password.RawPassword;
import io.inverno.mod.security.authentication.user.User;
import io.inverno.mod.security.identity.PersonIdentity;
import java.io.IOException;
import java.util.List;
import java.util.function.Supplier;
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;
import reactor.core.publisher.Mono;

public class TicketApp {

	private static final Logger LOGGER = LogManager.getLogger(TicketApp.class);

	public static final String REDIS_KEY = "APP:Ticket";
	public static final String PROFILE_PROPERTY_NAME = "profile";

	@Bean(name = "configurationSource")
	public interface TicketAppConfigurationSource extends Supplier<ConfigurationSource> {}

	@Bean(name = "configurationParameters")
	public interface TicketAppConfigurationParameters extends Supplier<List<ConfigurationKey.Parameter>> {}

	public static void main(String[] args) throws IOException {
		final BootstrapConfigurationSource bootstrapConfigurationSource = new BootstrapConfigurationSource(TicketApp.class.getModule(), args);
		Ticket ticketApp = bootstrapConfigurationSource
			.get(PROFILE_PROPERTY_NAME)
			.execute()
			.single()
			.map(configurationQueryResult -> configurationQueryResult.asString("default"))
			.map(profile -> {
				LOGGER.info(() -> "Active profile: " + profile);
				return Application.run(new Ticket.Builder()
					.setConfigurationSource(bootstrapConfigurationSource)
					.setConfigurationParameters(List.of(ConfigurationKey.Parameter.of(PROFILE_PROPERTY_NAME, profile)))
				);
			})
			.block();

		ticketApp.userRepository().getUser("jsmith")
			.switchIfEmpty(Mono.defer(() -> ticketApp.userRepository().createUser(User.of("jsmith")
				.password(new RawPassword("password"))
				.identity(new PersonIdentity("jsmith", "John", "Smith", "jsmith@inverno.io"))
				.build())
			))
			.block();
	}
}
```

You might have noticed that the password is specified in clear text as a `RawPassword`, but it is actually stored securely in the Redis data store using [Password-Based Key Derivation Function 2][pbkdf2] by default. This can be changed by defining a different password encoder in the `RedisUserRepository`.

> The user repository can also be exposed in a Web controller in order easily manage users from the application UI.

You should now be able to run the application using [Docker Compose][docker-compose] as described in the [full-stack application Guide][inverno-full-stack-guide] or directly using [Docker][docker] to start a Redis datastore and the Inverno Maven plugin to run the application.

```plaintext
$ docker run -d -p6379:6379 redis
```

```plaintext
$ mvn inverno:run
...
[INFO] --- inverno-maven-plugin:${VERSION_INVERNO_TOOLS}:run (default-cli) @ inverno-ticket ---
 [═══════════════════════════════════════════════ 100 % ══════════════════════════════════════════════] Running project io.inverno.guide.ticket@1.0.0-SNAPSHOT...
2024-12-19 10:42:01,769 INFO  [main] i.i.g.t.TicketApp - Active profile: default
2024-12-19 10:42:01,880 INFO  [main] i.i.c.v.Application - Inverno is starting...


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


2024-12-19 10:42:01,893 INFO  [main] i.i.g.t.Ticket - Starting Module io.inverno.guide.ticket...
2024-12-19 10:42:01,893 INFO  [main] i.i.m.b.Boot - Starting Module io.inverno.mod.boot...
2024-12-19 10:42:02,224 INFO  [main] i.i.m.b.Boot - Module io.inverno.mod.boot started in 330ms
2024-12-19 10:42:02,224 INFO  [main] i.i.m.r.l.Lettuce - Starting Module io.inverno.mod.redis.lettuce...
2024-12-19 10:42:02,287 INFO  [main] i.i.m.r.l.Lettuce - Module io.inverno.mod.redis.lettuce started in 62ms
2024-12-19 10:42:02,288 INFO  [main] i.i.m.s.j.Jose - Starting Module io.inverno.mod.security.jose...
2024-12-19 10:42:02,402 INFO  [main] i.i.m.s.j.Jose - Module io.inverno.mod.security.jose started in 114ms
2024-12-19 10:42:02,403 INFO  [main] i.i.m.w.s.Server - Starting Module io.inverno.mod.web.server...
2024-12-19 10:42:02,403 INFO  [main] i.i.m.h.s.Server - Starting Module io.inverno.mod.http.server...
2024-12-19 10:42:02,403 INFO  [main] i.i.m.h.b.Base - Starting Module io.inverno.mod.http.base...
2024-12-19 10:42:02,408 INFO  [main] i.i.m.h.b.Base - Module io.inverno.mod.http.base started in 4ms
2024-12-19 10:42:02,409 INFO  [main] i.i.m.w.b.Base - Starting Module io.inverno.mod.web.base...
2024-12-19 10:42:02,409 INFO  [main] i.i.m.h.b.Base - Starting Module io.inverno.mod.http.base...
2024-12-19 10:42:02,409 INFO  [main] i.i.m.h.b.Base - Module io.inverno.mod.http.base started in 0ms
2024-12-19 10:42:02,410 INFO  [main] i.i.m.w.b.Base - Module io.inverno.mod.web.base started in 1ms
2024-12-19 10:42:02,461 INFO  [main] i.i.m.h.s.i.HttpServer - HTTP Server (epoll) listening on http://0.0.0.0:8080
2024-12-19 10:42:02,462 INFO  [main] i.i.m.h.s.Server - Module io.inverno.mod.http.server started in 58ms
2024-12-19 10:42:02,463 INFO  [main] i.i.m.w.s.Server - Module io.inverno.mod.web.server started in 59ms
2024-12-19 10:42:02,683 INFO  [main] i.i.g.t.Ticket - Module io.inverno.guide.ticket started in 797ms
2024-12-19 10:42:02,684 INFO  [main] i.i.c.v.Application - Application io.inverno.guide.ticket started in 911ms
```

Now if you try to access to the application by opening [http://localhost:8080](http://localhost:8080) in your Web browser, you should be automatically redirected to the login page:

<img src="img/inverno_login_page.png" style="display: block; margin: 2em auto;" alt="Inverno Login page"/>

In order to access the application, you must enter valid login credentials, for instance `jsmith/password`, you should get redirected to the original request `/` and user's identity should be displayed in the UI:

<img src="img/inverno_ticket_authenticated.png" style="display: block; margin: 2em auto;" alt="Inverno Ticket application"/>

Once authenticated the user's identity can be retrieved by requesting [http://localhost:8080/api/security/identity](http://localhost:8080/api/security/identity):

```plaintext
$ curl -i -H "cookie: AUTH-TOKEN=eyJhbGciOiJIUzUxMiIsImtpZCI6InRrdCJ9.eyJpZGVudGl0eSI6eyJAYyI6Ii5QZXJzb25JZGVudGl0eSIsInVpZCI6ImpzbWl0aCIsImZpcnN0TmFtZSI6IkpvaG4iLCJsYXN0TmFtZSI6IlNtaXRoIiwiZW1haWwiOiJqc21pdGhAaW52ZXJuby5pbyJ9LCJ1c2VybmFtZSI6ImpzbWl0aCIsImF1dGhlbnRpY2F0ZWQiOnRydWUsImFub255bW91cyI6ZmFsc2UsImdyb3VwcyI6W119.mz2ez72LFj-m9flJ181kczlUgyTvDhJc64s4vfyKy8_6DHDSkMFp5jo_C3eydrOM-o3hwvA8IjCdGaK2P9ZAOw" http://localhost:8080/api/security/identity
HTTP/1.1 200 OK
content-type: application/json
content-length: 105

{"@c":".PersonIdentity","uid":"jsmith","firstName":"John","lastName":"Smith","email":"jsmith@inverno.io"}
```

Congratulations! You've just secured access to an Inverno Web application using a form-based login and JSON Web signature tokens.
