---
layout: doc
title: Clients&#58;
---

A [`Client`](https://github.com/pac4j/pac4j/blob/master/pac4j-core/src/main/java/org/pac4j/core/client/Client.java) represents an **authentication mechanism**. It performs the login process and returns (if successful) a user profile ([`CommonProfile`](https://github.com/pac4j/pac4j/blob/master/pac4j-core/src/main/java/org/pac4j/core/profile/CommonProfile.java)):

- [CAS protocol support](/docs/clients/cas.html)
- [SAML protocol support](/docs/clients/saml.html)
- [OpenID Connect protocol support](/docs/clients/openid-connect.html)
- [OAuth protocol support](/doc/clients/oauth.html)
- [HTTP protocol support](/docs/clients/http.html)
- [OpenID](/docs/clients/openid.html)
- [Google App Engine support](/docs/google-app-engine.html)


---

## `Client` methods

The `Client` interface has the following methods:

| Method | Usage |
|--------|-------|
| `HttpAction redirect(WebContext context) throws HttpAction` | redirects the user to the identity provider for login |
| `C getCredentials(WebContext context) throws HttpAction` | extracts the credentials from the HTTP request and validates them |
| `U getUserProfile(C credentials, WebContext context) throws HttpAction` | builds the authenticated user profile |

To define the security configuration of the application, all clients are defined in the [`Clients`](https://github.com/pac4j/pac4j/blob/master/pac4j-core/src/main/java/org/pac4j/core/client/Clients.java) object, then or directly in the [`Config`](https://github.com/pac4j/pac4j/blob/master/pac4j-core/src/main/java/org/pac4j/core/config/Config.java) object.

## Direct vs indirect clients

Clients are of two kinds:

| | Direct clients | Indirect clients
|------|----------------|-----------------
| Use cases | For web services | For web applications (UI)
| [Authentication flows](/docs/authentication-flows.html) | 1) Credentials are passed for each HTTP request (to the "[security filter](/docs/how-to-implement-pac4j-for-a-new-framework.html)") | 1) The originally requested url is saved in session (by the "security filter")<br />2) The user is redirected to the identity provider (by the "security filter")<br />3) Authentication happens at the identity provider (or locally for the `FormClient` and the `IndirectBasicAuthClient`)<br />4) The user is redirected back to the callback endpoint / url ("callback filter")<br />5) The user is redirected to the originally requested url (by the "[callback filter](/docs/how-to-implement-pac4j-for-a-new-framework.html)") |
| How many times the login process occurs? | The authentication happens for every HTTP request (in the "security filter") via the defined [`Authenticator`](/dcs/authenticators.html) and `ProfileCreator`.<br />For performance reasons, a cache may be used by wrapping the current `Authenticator` in a `LocalCachingAuthenticator` or the user profile can be saved (by the `Authenticator` or `ProfileCreator`) into the web session using the available web context and the `ProfileManager` class | The authentication happens only once (in the "callback filter") |
| Where is the user profile saved by default? | In the HTTP request  (stateless) | In the web session (statefull) |
| Where are the credentials? | Passed for every HTTP request (processed by the "security filter") | On the callback endpoint returned by the identity provider (and retrieved by the "callback filter") |
| Are the credentials mandatory? | Generally, no. If no credentials are provided, the direct client will be ignored (by the "security filter") | Generally, yes. Credentials are expected on the callback endpoint |
| What are the protected urls? | The urls of the web service are protected by the "security filter" | The urls of the web application are protected by the "security filter", but the callback url is not protected as it is used during the login process when the user is still anonymous |

---

## Compute roles and permissions

To compute the appropriate roles and permissions of the authenticated user profile, you need to define an [`AuthorizationGenerator`](https://github.com/pac4j/pac4j/blob/master/pac4j-core/src/main/java/org/pac4j/core/authorization/generator/AuthorizationGenerator.java) and attach it to the client.

**Example:**

```java
AuthorizationGenerator authGen = profile -> {
  String roles = profile.getAttribute("roles");
  for (String role: roles.split(",")) {
    profile.addRole(role);
  }
};
client.addAuthorizationGenerator(authGen);
```

You can attach it to the `Clients` object (in the `Config`) if you want the `AuthorizationGenerator` to be used for all clients.

---

## Custom `AjaxRequestResolver` and `CallbackUrlResolver`

For an indirect client, you define the callback url which will be used in the login process: after a successful login, the identity provider will redirect the user back to the application on the callback url.

By default, the callback url is expected to be an absolute url and is passed "as is" (by the `DefaultCallbackUrlResolver`). Though, you can provide your own [`CallbackUrlResolver`](https://github.com/pac4j/pac4j/blob/master/pac4j-core/src/main/java/org/pac4j/core/http/CallbackUrlResolver.java) to compute the appropriate callback url. The available [`RelativeCallbackUrlResolver`](https://github.com/pac4j/pac4j/blob/master/pac4j-core/src/main/java/org/pac4j/core/http/RelativeCallbackUrlResolver.java) turns the defined relative callback url into an absolute one. Example: `client.setCallbackUrlResolver(myCallbackUrlResolver);`.

For an indirect client, if the user tries to access a protected url, he will be redirected to the identity provider for login. Though, if the incoming HTTP request is an AJAX one, no redirection will be performed and a 401 error page will be returned. The HTTP request is considered to be an AJAX one if the value of the `X-Requested-With` header is `XMLHttpRequest` or if the `is_ajax_request` parameter or header is `true`. This is the behaviour of the [`DefaultAjaxRequestResolver`](https://github.com/pac4j/pac4j/blob/master/pac4j-core/src/main/java/org/pac4j/core/http/DefaultAjaxRequestResolver.java), but you can provide your own [`AjaxRequestResolver`](https://github.com/pac4j/pac4j/blob/master/pac4j-core/src/main/java/org/pac4j/core/http/AjaxRequestResolver.java). Example: `client.setAjaxRequestResolver(myAjaxRequestResolver);`.