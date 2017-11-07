---
layout:     series
title:      Securing a Spring RESTful Web Service with JWT
date:       2017-11-07 13:52:46 +0000
categories: tutorial
tags:       java maven spring-boot rest jwt security
section:    series
author:     juliuskrah
repo:       rest-example/tree/spring-jwt
---
> JSON Web Tokens ([JWT][]{:target="_blank"}) are an open, industry standard method for representing claims
  securely between two parties.

# Introduction
In a [previous post]({% post_url 2017-07-23-developing-restful-services-with-spring %}) we developed a REST 
Web Service with [Spring]({% post_url 2016-12-10-introduction-to-spring-and-dependency-injection-jsr-330 %}).
In the post I talked about how you can implement a REST Web Service with Spring. We also talked about the various
HTTP methods and how to map them to Spring's `RequestMapping` composite annotations.
In the post however, we did not talk about how to secure a Web Service. Security is an important aspect of 
building a Web Service and more often than not, it is overlooked.
In this post we are going to secure the web service using JWT with [Spring Security][]{:target="_blank"}.

Spring Security is a framework that focuses on providing both authentication and authorization to Java 
applications. To get started with this post get the source for the provious post as [zip](https://github.com/juliuskrah/rest-example/archive/v1.zip)|[tar.gz](hhttps://github.com/juliuskrah/rest-example/archive/v1.tar.gz) and extract the contents of the archive.

## Project Structure
At the end of this guide our folder structure will look similar to the following:

```
.
|__src/
|  |__main/
|  |  |__java/
|  |  |  |__com/
|  |  |  |  |__juliuskrah/
|  |  |  |  |  |__Application.java
|  |  |  |  |  |__Resource.java
|  |  |  |  |  |__ResourceService.java
|  |  |  |  |  |__security/
|  |  |  |  |  |  |__JWTAccessDeniedHandler.java
|  |  |  |  |  |  |__JWTAuthenticationEntryPoint.java
|  |  |  |  |  |  |__SecurityConfig.java
|  |  |  |  |  |  |__jwt/
|  |  |  |  |  |  |  |__JWTAuthenticationFilter.java
|  |  |  |  |  |  |  |__JWTConfigurer.java
|  |  |  |  |  |  |  |__JWTFilter.java
|  |  |  |  |  |  |  |__TokenProvider.java
|__pom.xml
```


# Prerequisites
To follow along this guide, your development system should have the following setup:

- [Java Development Kit][JDK]{:target="_blank"}  
- [Maven][]{:target="_blank"}
- [cURL][]{:target="_blank"}

# JWT: An Introduction
JSON Web Token (JWT) is an open standard (RFC 7519) that defines a compact and self-contained way for securely
transmitting information between parties as a JSON object. This information can be verified and trusted because 
it is digitally signed. JWTs can be signed using a secret (with the **HMAC**+ algorithm) or a public/private key 
pair using **RSA**.

JSON Web Tokens consist of three parts separated by dots (`.`), which are:
- Header
- Payload
- Signature

Therefore, a JWT typically looks like the following.  
`xxxxx.yyyyy.zzzzz`  
You can find a breakdown of the different parts on the [Official Website](https://jwt.io/introduction/){:target="_blank"}.

## Authentication Schemes
Security in a Web Application usually boils down Authentication and Authorization. There are several 
[`shemes`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Authentication#Authentication_schemes){:target="_blank"} 
involved when doing Authentication and Authorization. Some of the common ones are listed below:
- Basic (RFC 7617, base64-encoded credentials.),
- Bearer (RFC 6750, bearer tokens to access OAuth 2.0-protected resources),
- Digest (RFC 7616, only md5 hashing is supported in Firefox),
- HOBA (RFC 7486 (draft), **H**TTP **O**rigin-**B**ound **A**uthentication, digital-signature-based),
- Mutual (draft-ietf-httpauth-mutual),
- AWS4-HMAC-SHA256 (AWS docs).

IANA maintains a list of authentication schemes [here](http://www.iana.org/assignments/http-authschemes/http-authschemes.xhtml){:target="_blank"}.
We will use the `Bearer` authentication scheme in this post.

## Add Dependencies for Security
We will add the dependencies for `JWT` and `Spring Security`:

file: {% include file-path.html file_path='pom.xml' %}

{% highlight xml %}
...
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-security</artifactId>
</dependency>
<dependency>
  <groupId>io.jsonwebtoken</groupId>
  <artifactId>jjwt</artifactId>
  <version>0.9.0</version>
</dependency>
{% endhighlight %}

Since we are leveraging the power of Spring-Boot, the above dependency to Spring Security registers the Spring
Security Filter Chain and sets up the `Basic` Authentication Scheme.  
Open a terminal and run the project:

{% highlight bash %}
> mvnw clean spring-boot:run
{% endhighlight %}

Open another terminal and run:

{% highlight bash %}
> curl -i -H "Accept: application/json" http://localhost:8080/api/v1.0/resources

HTTP/1.1 401
X-Content-Type-Options: nosniff
X-XSS-Protection: 1; mode=block
Cache-Control: no-cache, no-store, max-age=0, must-revalidate
Pragma: no-cache
Expires: 0
X-Frame-Options: DENY
Strict-Transport-Security: max-age=31536000 ; includeSubDomains
WWW-Authenticate: Basic realm="Spring"
Content-Type: application/json;charset=UTF-8
Transfer-Encoding: chunked
Date: Tue, 07 Nov 2017 10:10:16 GMT

{
  "timestamp":"2017-11-07T10:10:16.665+0000",
  "status":401,
  "error":"Unauthorized",
  "message":"Full authentication is required to access this resource",
  "path":"/api/v1.0/resources"
}
{% endhighlight %}

As you can see, you got an `Access Denied` when you tried to access a protected resource. Let's create a password
and username and try again:

file: {% include file-path.html file_path='src/main/resources/application.properties' %}

{% highlight properties %}
security.user.name=julius
security.user.password=secret
{% endhighlight %}

{% highlight bash %}
> curl -i -H "Accept: application/json" http://localhost:8080/api/v1.0/resources -ujulius:secret

HTTP/1.1 200
X-Content-Type-Options: nosniff
X-XSS-Protection: 1; mode=block
Cache-Control: no-cache, no-store, max-age=0, must-revalidate
Pragma: no-cache
Expires: 0
X-Frame-Options: DENY
Strict-Transport-Security: max-age=31536000 ; includeSubDomains
Content-Type: application/json;charset=UTF-8
Transfer-Encoding: chunked
Date: Tue, 07 Nov 2017 10:21:47 GMT

[
  {
    "id":1,
    "description":"Resource One",
    "createdTime":"2017-11-07T10:21:23.049",
    "modifiedTime":null
  },
  {
    "id":2,
    "description":"Resource Two",
    "createdTime":"2017-11-07T10:21:23.049",
    "modifiedTime":null
  }
  ...
]
{% endhighlight %}

The `-u` switch is to inform curl we intend to authenticate with a username:password combination. You can also request with the `Base64` encoding:

{% highlight bash %} 
> curl -i -H "Accept: application/json" -H "Authorization: Basic anVsaXVzOnNlY3JldA==" http://localhost:8080/api/v1.0/resources
{% endhighlight %}


## JWT and Bearer Authentication
Now that we have seen the `Basic` scheme in action, let us now switch to the `Bearer` scheme using JWT.
Create the Spring Security `WebSecurityConfigurerAdapter` class:

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/security/SecurityConfig.java' %}

{% highlight java %}
@Configuration
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {

  private final AuthenticationManagerBuilder auth;
  private final TokenProvider tokenProvider;
  ...

  public SecurityConfig(AuthenticationManagerBuilder auth, TokenProvider tokenProvider) {
    //
  }

  @PostConstruct
  public void init() {
    try {
      auth
        .inMemoryAuthentication()
          .withUser("julius").password("secret").roles("USER")
        .and()
          .withUser("admin").password("admin").roles("USER", "ADMIN");
    } catch (Exception e) {
      throw new BeanInitializationException("Security configuration failed", e);
    }
  } 

  @Override
  protected void configure(HttpSecurity http) throws Exception {
    http
      .addFilterBefore(new JWTAuthenticationFilter(
                "/api/authenticate", this.tokenProvider, authenticationManager()),  
                UsernamePasswordAuthenticationFilter.class)          //  <1>
      .exceptionHandling()
        .authenticationEntryPoint(new JWTAuthenticationEntryPoint()) //  <2>
        .accessDeniedHandler(new JWTAccessDeniedHandler())           //  <3>
      .and()
        .csrf()
        .disable()                                                   //  <4>
      .and()
        .sessionManagement()
        .sessionCreationPolicy(SessionCreationPolicy.STATELESS)      //  <5>
      .and()
        .authorizeRequests().anyRequest().authenticated()            //  <6>
      .and()
        .apply(securityConfigurerAdapter())                          //  <7>
    }
}
{% endhighlight %}

1. This is to ensure that the Authentication filter is called before the `UsernamePasswordAuthenticationFilter`.
   Once this is registered first, this will be called before the other authentication filters in the Spring
   Security Filter Chain. This filter will check for a valid login before proceeding. All login attempts will
   be a POST request to `/api/authenticate/`.
2. The `AuthenticationEntryPoint` will ensure users are presented with an `Unauthorized` 401 response. This will
   override the Spring Security default of redirecting to a login page. Login pages do not make sense for REST
   services.
3. `AccessDeniedHandler` for when a user tries to access a resource he is not permitted to access.
4. We will disable `CSRF` protection. It is not required in REST.
5. We will set session management to `STATELESS` for REST is stateless.
6. Make sure all routes require authentication.
7. Apply a security configurer to authenticate all requests.

Create a class that takes care of the `concern` of enconding and decoding the JWTs:

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/security/jwt/TokenProvider.java' %}

{% highlight java %}
@Component
public class TokenProvider {
  private static final String AUTHORITIES_KEY = "auth";
  private String secretKey = "secret";
  private long tokenValidityInMilliseconds = 1000 * 86400;

  public String createToken(Authentication authentication) {
    // Click on link above for implementation
  }

  public Authentication getAuthentication(String token) {
    // Decode token
  }

  public boolean validateToken(String authToken) {
    // Return true if token is valid
  }
}
{% endhighlight %}

We need a filter class to filter all requests for the `Bearer` token:

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/security/jwt/JWTFilter.java' %}

{% highlight java %}
public class JWTFilter extends GenericFilterBean {
  private final TokenProvider tokenProvider;

  public JWTFilter(TokenProvider tokenProvider) {
    this.tokenProvider = tokenProvider;
  }

  @Override
  public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain)
            throws IOException, ServletException {
    // Filter each request for the Bearer token      
  }

  private String resolveToken(HttpServletRequest request) {
    // Resolve token
  }
}
{% endhighlight %}

We put it all together in `JWTConfigurer` that will encapsulate each request:

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/security/jwt/JWTConfigurer.java' %}

{% highlight java %}
public class JWTConfigurer extends SecurityConfigurerAdapter<DefaultSecurityFilterChain, HttpSecurity> {
  public static final String AUTHORIZATION_HEADER = "Authorization";
  public static final String AUTHORIZATION_TOKEN = "access_token";
  private TokenProvider tokenProvider;

  public JWTConfigurer(TokenProvider tokenProvider) {
    this.tokenProvider = tokenProvider;
  }

  @Override
  public void configure(HttpSecurity http) throws Exception {
    JWTFilter customFilter = new JWTFilter(tokenProvider);
    http.addFilterBefore(customFilter, UsernamePasswordAuthenticationFilter.class);
  }
}
{% endhighlight %}

With all this done, we need to be able to login to generate the token that will be used for subsequent requests:

file: {% include file-path.html file_path='src/main/java/com/juliuskrah/security/jwt/JWTAuthenticationFilter.java' %}

{% highlight java %}
public class JWTAuthenticationFilter extends AbstractAuthenticationProcessingFilter {
  private final TokenProvider tokenProvider;
  private final AuthenticationManager authenticationManager;

  public JWTAuthenticationFilter(String defaultFilterProcessesUrl, TokenProvider tokenProvider,
                                   AuthenticationManager authenticationManager) {
    super(new AntPathRequestMatcher(defaultFilterProcessesUrl));
    this.tokenProvider = tokenProvider;
    this.authenticationManager = authenticationManager;
  }

  @Override
  public Authentication attemptAuthentication(HttpServletRequest request, HttpServletResponse response)
            throws AuthenticationException, IOException, ServletException {
    // Do authentication
  }

  @Override
  protected void successfulAuthentication(HttpServletRequest request, HttpServletResponse response,
                                            FilterChain chain, Authentication authentication) throws IOException, ServletException {
    // Add Bearer token to header
  }
}
{% endhighlight %}

We can test what we did now:

{% highlight bash %}
> curl -X POST -i -H "Accept: application/json" -H "Content-Type: application/x-www-form-urlencoded" http://localhost:8080/api/authenticate -d "username=julius&password=secret"

HTTP/1.1 200
X-Content-Type-Options: nosniff
X-XSS-Protection: 1; mode=block
Cache-Control: no-cache, no-store, max-age=0, must-revalidate
Pragma: no-cache
Expires: 0
Authorization: Bearer eyJhbGciOiJIUzUxMiJ9.eyJzdWIiOiJqdWxpdXMiLCJhdXRoIjoiUk9MRV9VU0VSIiwiZXhwIjoxNTEwMTgwMDg4fQ.taul450j0s39mnyzfdd3jurloRUlCowJK6Vd6EmN2tv5y9iiGTfPXyfLBDx0NLOZKUHcbhgYSUe_AoK_DzybGg
Content-Length: 0
Date: Tue, 07 Nov 2017 11:05:14 GMT
{% endhighlight %}

Notice the `Authorization` header with the Bearer token 
```
eyJhbGciOiJIUzUxMiJ9.eyJzdWIiOiJqdWxpdXMiLCJhdXRoIjoiUk9MRV9VU0VSIiwiZXhwIjoxNTEwMTgwMDg4fQ.taul450j0s39mnyzfdd3jurloRUlCowJK6Vd6EmN2tv5y9iiGTfPXyfLBDx0NLOZKUHcbhgYSUe_AoK_DzybGg
```
We will use this token to access all other resources in our service:

{% highlight bash %}
> curl -i -H "Accept: application/json" -H "Authorization: Bearer eyJhbGciOiJIUzUxMiJ9.eyJzdWIiOiJqdWxpdXMiLCJhdXRoIjoiUk9MRV9VU0VSIiwiZXhwIjoxNTEwMTgwMDg4fQ.taul450j0s39mnyzfdd3jurloRUlCowJK6Vd6EmN2tv5y9iiGTfPXyfLBDx0NLOZKUHcbhgYSUe_AoK_DzybGg" http://localhost:8080/api/v1.0/resources

HTTP/1.1 200
X-Content-Type-Options: nosniff
X-XSS-Protection: 1; mode=block
Cache-Control: no-cache, no-store, max-age=0, must-revalidate
Pragma: no-cache
Expires: 0
Content-Type: application/json;charset=UTF-8
Transfer-Encoding: chunked
Date: Tue, 07 Nov 2017 11:34:46 GMT

[
  {
    "id":1,
    "description":"Resource One",
    "createdTime":"2017-11-07T11:27:36.175",
    "modifiedTime":null
  },
  {
    "id":2,
    "description":"Resource Two",
    "createdTime":"2017-11-07T11:27:36.175",
    "modifiedTime":null
  }
  ...
]
{% endhighlight %}

You can play around with it. Change some values and see how it behaves.  
That's all folks.

# Conclusion
In this post we learnt how to secure a RESTful web service with JWT and Spring Security.

As usual you can find the full example to this guide {% include source.html %}. Until the next post, keep doing cool things :+1:.

[cURL]:                     https://curl.haxx.se/
[JDK]:                      http://www.oracle.com/technetwork/java/javase/downloads/index.html
[JWT]:                      https://jwt.io/
[Maven]:                    http://maven.apache.org/
[Spring Security]:          https://projects.spring.io/spring-security/