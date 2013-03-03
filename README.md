# An OAuth 2 Endpoint Plugin for Restify

This package provides a *very simple* plugin for the [Restify][] framework, giving your RESTful server OAuth 2.0
endpoint capabilities. In particular, it implements the [Resource Owner Password Credentials flow][ropc] only.

TODO: yeah we're gonna need to explain ROPC and bearer tokens and shit. Maybe I'll make a blog post and link to that?
GREAT IDEA!

## What You Get

TODO: Explain the hooks first, rename this section.

If you provide this plugin with the appropriate hooks, it will:

* Set up a [token endpoint][], which returns [access token responses][token-endpoint-success] or
  [correctly-formatted error responses][token-endpoint-error]. It will accept either
  `"application/x-www-form-urlencoded"` as specified, or `"application/json"` as described in some RFC I can't find
  anymore. Client validation and access token generation is left up to your hooks.
* Protect all other resources that you specify, verifying that a bearer token is sent and that it passes your
  authentication hook.
  * If it does, the request object will get the `username` property returned from your authentication hook.
  * Otherwise, a 401 error will be sent with an appropriate `"WWW-Authenticate"` header as well as a
    [`"Link"` header][web-linking] with [`rel="oauth2-token"`][oauth2-token-rel] pointing to the token endpoint.

## Use and Configuration

To use Restify-OAuth2, you'll need to instantiate a new instance of the plugin and call `server.use` on your Restify
server. Restify-OAuth2 also depends on the built-in `authorizationParser` and `bodyParser` plugins. So in short, it
looks like this:

```js
var restify = require("restify");
var restifyOAuth2 = require("restify-oauth2");

var server = restify.createServer({ name: "My cool server", version: "1.0.0" });
server.use(restify.authorizationParser());
server.use(restify.bodyParser({ mapParams: false }));
server.use(restifyOAuth2(options));
```

TODO explain how to check `req.username` or similar.

Maybe restructure to add a separate plugin (e.g. `restifyOAuth2.denyAccess(requiresTokenPredicate)`) for that aspect?

### Hooks

To hook Restify-OAuth2 up to your infrastructure, you will need to provide it with the following hooks in the `options`
hash:

* `validateClient(clientId, clientSecret, cb)`: should check that the API client is valid.
* `grantToken(username, password, cb)`: given a username and password for the user the client is connecting on behalf
  of, should check that the user authenticates, and if so, generate and store a token for them.
* `authenticateToken(token, cb)`: given a token, should check to make sure it has been granted, and if so translated it
  to a username.

Basically, if you can provide those, you get the OAuth2 implementation for free. Here are some [example hooks][].

### Other Options

The hooks above are the only required options, but the following are also available for tweaking:

* `tokenEndpoint`: the location at which the token endpoint should be created. Defaults to `"/token"`.
* `wwwAuthenticateRealm`: the value of the "Realm" challenge in the [`"WWW-Authenticate"`][www-authenticate] header.
  Defaults to `"Who goes there?"`.
* `tokenExpirationTime`: the value returned for the `expires_in` component of the response from the token endpoint.
  Note that this is *only* the value reported; you are responsible for keeping track of token expiration yourself.
  Defaults to `Infinity` (which results in no value being sent in the response).
* `requiresTokenPredicate`: a function which takes a request object, and returns whether it requires a token to access.
  This can be used to create public resources that can be accessed without a token. Defaults to a function that returns
  `true` for all requests except those to the root (initial) resource.

## What Does That Look Like?

OK, let's try something a bit more concrete. If you check out the [example server][] used in the integration tests,
you'll see our setup:

* An initial resource, at `/`.
  * It always contains a link to the public resource at `/public`.
  * If no token is supplied, the token endpoint is linked to at `/token`.
  * If a token is supplied and has been validated, as shown by the presence of `req.username`, the secret resource is
    linked to at `/secret`.
* A `/token` resource, automatically provided by Restify-OAuth2, which is used to generate tokens for a given client ID/
  client secret/username/password combination. The client validation and token-generation logic is provided by the
  application, but none of the ceremony necessary for OAuth 2 conformance is present in the application code:
  Restify-OAuth2 takes care of all of that.
* A `/public` resource that anyone can access.
  * If a token is supplied, it will be validated by Restify-OAuth2, and a `username` property attached to the request
    object. In particular, a token that does not pass authentication will result in a 401 error.
  * But if no token is supplied, that's OK: Restify-OAuth2 simply passes the request along to the application, which
    shows the contents of the public resource.
* A `/secret` resource that requires a bearer token.
  * If none is supplied, or one is supplied but it doesn't authenticate, a 401 response will be generated by
    Restify-OAuth2, with `"WWW-Authenticate"` and `"Link"` headers as above.
  * If one is supplied, the application displays the contents of the resource.

TODO: update example public page to use `req.username` to say "hi, username!" if logged in, or "sign up here!" if not.

[Restify]: http://mcavage.github.com/node-restify/
[ropc]: http://tools.ietf.org/html/draft-ietf-oauth-v2-30#section-1.3.3
[token endpoint]: http://tools.ietf.org/html/draft-ietf-oauth-v2-30#section-4.3.2
[token-endpoint-success]: http://tools.ietf.org/html/draft-ietf-oauth-v2-30#section-5.1
[token-endpoint-error]: http://tools.ietf.org/html/draft-ietf-oauth-v2-30#section-5.2
[oauth2-token-rel]: http://tools.ietf.org/html/draft-wmills-oauth-lrdd-01#section-4.1.2
[web-linking]: http://tools.ietf.org/html/rfc5988
[www-authenticate]: http://tools.ietf.org/html/rfc2617#section-3.2.1
[example hooks]: https://github.com/domenic/restify-oauth2/blob/master/examples/hooks.js
[example server]: https://github.com/domenic/restify-oauth2/blob/master/examples/server.js
