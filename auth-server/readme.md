一、
1、An Authorization Server is nothing more than a bunch of endpoints, and they are implemented in Spring OAuth2 as Spring MVC handlers. We already have a secure application, so it’s really just a matter of adding the @EnableAuthorizationServer annotation： com.elasticjee.SocialApplication

  with that new annotation in place Spring Boot will install all the necessary endpoints and set up the security for them, provided we supply a few details of an OAuth2 client we want to support:
```yaml
security:
  oauth2:
    client:
      client-id: acme
      client-secret: acmesecret
      scope: read,write
      auto-approve-scopes: '.*'
```

This client is the equivalent of the facebook.client* and github.client* that we need for the external authentication. With the external providers we had to register and get a client ID and a secret to use in our app. In this case we are providing our own equivalent of the same feature, so we need (at least one) client for it to work.

We have set the auto-approve-scopes to a regex matching all scopes. This is not necessarily where we would leave this app in a real system, but it gets us something working quickly without having toreplace the whitelabel approval page that Spring OAuth2 would otherwise pop up for our users when they wanted an access token. To add an explicit approval step to the token grant we would need to provide a UI replacing the whitelabel version (at /oauth/confirm_access).

We have set the auto-approve-scopes to a regex matching all scopes. This is not necessarily where we would leave this app in a real system, but it gets us something working quickly without having toreplace the whitelabel approval page that Spring OAuth2 would otherwise pop up for our users when they wanted an access token. To add an explicit approval step to the token grant we would need to provide a UI replacing the whitelabel version (at /oauth/confirm_access).

2、To finish the Authorization Server we just need to provide security configuration for its UI. In fact there isn’t much of a user interface in this simple app, but we still need to protect the /oauth/authorize endpoint, and make sure that the home page with the "Login" buttons is visible. That’s why we have this method:
``` java
@Override
protected void configure(HttpSecurity http) throws Exception {
  http.antMatcher("/**")                                       (1)
    .authorizeRequests()
      .antMatchers("/", "/login**", "/webjars/**").permitAll() (2)
      .anyRequest().authenticated()                            (3)
    .and().exceptionHandling()
      .authenticationEntryPoint(new LoginUrlAuthenticationEntryPoint("/")) (4)
    ...
}
```
1	All requests are protected by default
2	The home page and login endpoints are explicitly excluded
3	All other endpoints require an authenticated user
4	Unauthenticated users are re-directed to the home page

3、How to Get an Access Token

  Access tokens are now available from our new Authorization Server. The simplest way to get a token up to now is to grab one as the "acme" client. You can see this if you run the app and curl it:
```shell
$ curl acme:acmesecret@localhost:8080/oauth/token -d grant_type=client_credentials
{"access_token":"370592fd-b9f8-452d-816a-4fd5c6b4b8a6","token_type":"bearer","expires_in":43199,"scope":"read write"}
```

4、Client credentials tokens are useful in some circumstances (like testing that the token endpoint works), but to take advantage of all the features of our server we want to be able to create tokens for users. To get a token on behalf of a user of our app we need to be able to authenticate the user. If you were watching the logs carefully when the app started up you would have seen a random password being logged for the default Spring Boot user (per the Spring Boot User Guide(http://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#boot-features-security)). You can use this password to get a token on behalf of the user with id "user":
```shell
$ curl acme:acmesecret@localhost:8080/oauth/token -d grant_type=password -d username=user -d password=...
{"access_token":"aa49e025-c4fe-4892-86af-15af2e6b72a2","token_type":"bearer","refresh_token":"97a9f978-7aad-4af7-9329-78ff2ce9962d","expires_in":43199,"scope":"read write"}
```
where "…​" should be replaced with the actual password. This is called a "password" grant, where you exchange a username and password for an access token.

Password grant is also mainly useful for testing, but can be appropriate for a native or mobile application, when you have a local user database to store and validate the credentials. For most apps, or any app with "social" login, like ours, you need the "authorization code" grant, and that means you need a browser (or a client that behaves like a browser) to handle redirects and cookies, and render the user interfaces from the external providers.


二、Creating a Client Application
1、A client application for our Authorization Server that is itself a web application is easy to create with Spring Boot. Here’s an example:
```java
@EnableAutoConfiguration
@Configuration
@EnableOAuth2Sso
@RestController
public class ClientApplication {

  @RequestMapping("/")
  public String home(Principal user) {
    return "Hello " + user.getName();
  }

  public static void main(String[] args) {
    new SpringApplicationBuilder(ClientApplication.class)
        .properties("spring.config.name=client").run(args);
  }

}
```

2、The ingredients of the client are a home page (just prints the user’s name), and an explicit name for a configuration file (via spring.config.name=client). When we run this app it will look for a configuration file which we provide as follows:
```yaml
server:
  port: 9999
  context-path: /client
security:
  oauth2:
    client:
      client-id: acme
      client-secret: acmesecret
      access-token-uri: http://localhost:8080/oauth/token
      user-authorization-uri: http://localhost:8080/oauth/authorize
    resource:
      user-info-uri: http://localhost:8080/me
```

The configuration looks a lot like the values we used in the main app, but with the "acme" client instead of the Facebook or Github ones. The app will run on port 9999 to avoid conflicts with the main app.
### And it refers to a user info endpoint "/me" which we haven’t implemented yet.

Note that the server.context-path is set explicitly, so if you run the app to test it remember the home page is http://localhost:9999/client. Clicking on that link should take you to the auth server and once you you have authenticated with the social provider of your choice you will be redirected back to the client app

### The context path has to be explicit if you are running both the client and the auth server on localhost, otherwise the cookie paths clash and the two apps cannot agree on a session identifier.


3、Protecting the User Info Endpoint

To use our new Authorization Server for single sign on, just like we have been using Facebook and Github, it needs to have a /user endpoint that is protected by the access tokens it creates. So far we have a /user endpoint, and it is secured with cookies created when the user authenticates. To secure it in addition with the access tokens granted locally we can just re-use the existing endpoint and make an alias to it on a new path:
```java
@RequestMapping({ "/user", "/me" })
public Map<String, String> user(Principal principal) {
  Map<String, String> map = new LinkedHashMap<>();
  map.put("name", principal.getName());
  return map;
}
```
We have converted the Principal into a Map so as to hide the parts that we don’t want to expose to the browser, and also to unfify the behaviour of the endpoint between the two external authentication providers. In principle we could add more detail here, like a provider-specific unique identifier for instance, or an e-mail address if it’s available.

**The "/me" path** can now be protected with the access token by declaring that our app is a **Resource Server** (as well as an Authorization Server). We create a new configuration class (as n inner class in the main app, but it could also be split out into a separate standalone class):
```java
@Configuration
@EnableResourceServer
protected static class ResourceServerConfiguration
    extends ResourceServerConfigurerAdapter {
  @Override
  public void configure(HttpSecurity http) throws Exception {
    http
      .antMatcher("/me")
      .authorizeRequests().anyRequest().authenticated();
  }
}
```

In addition we need to specify an @Order for the main application security:
```java
@SpringBootApplication
...
@Order(SecurityProperties.ACCESS_OVERRIDE_ORDER)
public class SocialApplication extends WebSecurityConfigurerAdapter {
  ...
}
```

##### The @EnableResourceServer annotation creates a security filter with @Order(SecurityProperties.ACCESS_OVERRIDE_ORDER-1) by default, so by moving the main application security to @Order(SecurityProperties.ACCESS_OVERRIDE_ORDER) we ensure that the rule for "/me" takes precedence.

Testing the OAuth2 Client

4、Testing the OAuth2 Client
To test the new features you can just run both apps and visit http://localhost:9999/client in your browser. The client app will redirect to the local Authorization Server, which then gives the user the usual choice of authentication with Facebook or Github. Once that is complete control returns to the test client, the local access token is granted and authentication is complete (you should see a "Hello" message in your browser). If you are already authenticated with Github or Facebook you may not even notice the remote authentication.