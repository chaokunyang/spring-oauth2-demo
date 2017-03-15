1、There are 2 features behind @EnableOAuth2Sso: the OAuth2 client, and the authentication. The client is re-usable, so you can also use it to interact with the OAuth2 resources that your Authorization Server (in this case Facebook) provides (in this case the Graph API). The authentication piece aligns your app with the rest of Spring Security, so once the dance with Facebook is over your app behaves exactly like any other secure Spring app.

2、The client piece is provided by Spring Security OAuth2 and switched on by a different annotation @EnableOAuth2Client. So the first step in this transformation is to remove the @EnableOAuth2Sso and replace it with the lower level annotation: @EnableOAuth2Client

3、Once that is done we have some stuff created for us that will be useful. First off we can inject an OAuth2ClientContext and use it to build an authentication filter that we add to our security configuration:
``` java
@SpringBootApplication
@EnableOAuth2Client
@RestController
public class SocialApplication extends WebSecurityConfigurerAdapter {

  @Autowired
  OAuth2ClientContext oauth2ClientContext;

  @Override
  protected void configure(HttpSecurity http) throws Exception {
    http.antMatcher("/**")
      ...
      .addFilterBefore(ssoFilter(), BasicAuthenticationFilter.class);
  }

  ...

}
```

4、this filter is created in new method where we use the OAuth2ClientContext:
``` java
private Filter ssoFilter() {
  OAuth2ClientAuthenticationProcessingFilter facebookFilter = new OAuth2ClientAuthenticationProcessingFilter("/login/facebook");
  OAuth2RestTemplate facebookTemplate = new OAuth2RestTemplate(facebook(), oauth2ClientContext);
  facebookFilter.setRestTemplate(facebookTemplate);
  UserInfoTokenServices tokenServices = new UserInfoTokenServices(facebookResource().getUserInfoUri(), facebook().getClientId());
  tokenServices.setRestTemplate(facebookTemplate);
  facebookFilter.setTokenServices(tokenServices);
  return facebookFilter;
}
```

5、the filter also needs to know about the client registration with Facebook:
``` java
  @Bean
  @ConfigurationProperties("facebook.client")
  public AuthorizationCodeResourceDetails facebook() {
    return new AuthorizationCodeResourceDetails();
  }
```

6、and to complete the authentication it needs to know where the user info endpoint is in Facebook:
``` java
  @Bean
  @ConfigurationProperties("facebook.resource")
  public ResourceServerProperties facebookResource() {
    return new ResourceServerProperties();
  }
```

7、Note that with both these "static" data objects (facebook() and facebookResource()) we used a @Bean decorated as @ConfigurationProperties. That means that we can convert the application.yml to a slightly new format, where the prefix for configuration is facebook instead of security.oauth2:
``` yaml
facebook:
  client:
    clientId: 233668646673605
    clientSecret: 33b17e044ee6a4fa383f46ec6e28ea1d
    accessTokenUri: https://graph.facebook.com/oauth/access_token
    userAuthorizationUri: https://www.facebook.com/dialog/oauth
    tokenName: oauth_token
    authenticationScheme: query
    clientAuthenticationScheme: form
  resource:
    userInfoUri: https://graph.facebook.com/me
```
因为原先的
``` yaml
security:
  oauth2:
    client:
      clientId: 233668646673605
      clientSecret: 33b17e044ee6a4fa383f46ec6e28ea1d
      accessTokenUri: https://graph.facebook.com/oauth/access_token
      userAuthorizationUri: https://www.facebook.com/dialog/oauth
      tokenName: oauth_token
      authenticationScheme: query
      clientAuthenticationScheme: form
    resource:
      userInfoUri: https://graph.facebook.com/me
```
是Spring 的@EnableOauth2Sso的默认命名，@EnableOauth2Sso的默认读取security.oauth2下的配置，而现在我们是手动自己配置，所以命名可以更加自由。

8、Finally, we changed the path to the login to be facebook-specific in the Filter declaration above, so we need to make the same change in the HTML:
``` html
<h1>Login</h1>
<div class="container" ng-show="!home.authenticated">
	<div>
	With Facebook: <a href="/login/facebook">click here</a>
	</div>
</div>
```

9、Handling the Redirects

The last change we need to make is to explicitly support the redirects from our app to Facebook. This is handled in Spring OAuth2 with a servlet Filter, and the filter is already available in the application context because we used @EnableOAuth2Client. ALl that is needed is to wire the filter up so that it gets called in the right order in our Spring Boot application. To do that we need a FilterRegistrationBean:
``` java
@Bean
public FilterRegistrationBean oauth2ClientFilterRegistration(
    OAuth2ClientContextFilter filter) {
  FilterRegistrationBean registration = new FilterRegistrationBean();
  registration.setFilter(filter);
  registration.setOrder(-100);
  return registration;
}
```

We autowire the already available filter, and register it with a sufficiently low order that it comes before the main Spring Security filter. In this way we can use it to handle redirects signaled by exceptions in authentication requests.