1、The /logout endpoint requires us to POST to it, and to protect the user from Cross Site Request Forgery (CSRF, pronounced "sea surf"), it requires a token to be included in the request. The value of the token is linked to the current session, which is what provides the protection, so we need a way to get that data into our JavaScript app.

  AngularJS also has built in support for CSRF (they call it XSRF), but it is implemented in a slightly different way than the out-of-the box behaviour of Spring Security. What Angular would like is for the server to send it a cookie called "XSRF-TOKEN" and if it sees that, it will send the value back as a header named "X-XSRF-TOKEN". To teach Spring Security about this we need to add a filter that creates the cookie and also we need to tell the existing CRSF filter about the header name. In the WebSecurityConfigurer:

2、org.springframework.security.web.csrf.CsrfTokenRepository

A CsrfTokenRepository that persists the CSRF token in a cookie named "XSRF-TOKEN" and reads from the header "X-XSRF-TOKEN" following the conventions of AngularJS. When using with AngularJS be sure to use withHttpOnlyFalse().
