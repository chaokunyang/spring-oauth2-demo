基于spring-boot、spring-security、spring-security-oauth2的oauth2 demo

1、The apps all work on localhost:8080 because they use OAuth2 clients registered with Facebook and Github for that address. To run them on a different host or port, you need to register your own apps and put the credentials in the config files. There is no danger of leaking your Facebook or Github credentials beyond localhost if you use the default values, but be careful what you expose on the internet, and don’t put your own app registrations in public source control.

2、How to Add a Local User Database

  Many applications need to hold data about their users locally, even if authentication is delegated to an external provider. We don’t show the code here, but it is easy to do in two steps.

  Choose a backend for your database, and set up some repositories (e.g. using Spring Data) for a custom User object that suits your needs and can be populated, fully or partially, from the external authentication.

  Provision a User object for each unique user that logs in by inspecting the repository in your /user endpoint. If there is already a user with the identity of the current Principal, it can be updated, otherwise created.

  Hint: add a field in the User object to link to a unique identifier in the external provider (not the user’s name, but something that is unique to the account in the external provider).