## Kubernetes Access Control

#### Connecting Kubernetes to your Identity Provider with OpenID Connect

https://github.com/pvdvreede/kubernetes-auth-presentation

Note:
* Presentation is on Github if you want to refer back later or copy anything from it

---

#### Why?

* We want to stop developers remembering yet another password
* We want operators to not have to manage another user management system
* We want to tie into our company's onboarding and offboarding procedures to give and revoke access without extra work

Note:
* Why would anyone want to bother doing this?
* It is some extra complexity to learn and understand upfront
* But there are benefits longer term
* However it does put a dependency on a third party system that k8s operators may or may not control

---

#### Potential providers

* Google
* Twitter
* Github
* Microsoft ADFS 2016
* Auth0
* Okta

Note:
* There are many solutions for integrating with
* Most possible IdPs that a Company may use would most likely be compatible
* Essentially anything that supports OpenID Connect will work

---

#### How does Kubernetes Auth/RBAC work?

```
kubectl

  \/

authentication

  \/

authorization / RBAC

  \/

admission controllers
```

@[1]
@[2-5]
@[6-9]
@[10-13]

---

#### Kubernetes RBAC

* Resources and Verbs
* API Groups
* Cluster level resources (ClusterRole) v Namespace level (Role)
* Mixing and matching

---

#### What is OpenID Connect

* Sits on top of OAuth2 as an extension
* Dictates the claims being passed
* Has its own OAuth2 scope
* Provides introspection of the configuration via a `/.well-known/openidconfiguration` discovery endpoint

Note:
* The OAuth 2 spec specifies the auth flow and other things
* OpenID Connect just leverages that and standardises some of the claims present
* The well known endpoint contains uris, encryption keys so that they do not need to be specified by the client everytime and can be changed

---

#### openid discovery

```json
  {
   "issuer": "https://server.example.com",
   "authorization_endpoint":"https://server.example.com/connect/authorize",
   "token_endpoint":"https://server.example.com/connect/token",
   "token_endpoint_auth_methods_supported":["client_secret_basic", "private_key_jwt"],
   "token_endpoint_auth_signing_alg_values_supported":["RS256", "ES256"],
   "userinfo_endpoint":"https://server.example.com/connect/userinfo",
   "check_session_iframe":"https://server.example.com/connect/check_session",
   "end_session_endpoint":"https://server.example.com/connect/end_session",
   "jwks_uri":"https://server.example.com/jwks.json",
   "registration_endpoint":"https://server.example.com/connect/register",
   "scopes_supported":["openid", "profile", "email", "address","phone", "offline_access"],
   "response_types_supported":["code", "code id_token", "id_token", "token id_token"],
   "acr_values_supported":
     ["urn:mace:incommon:iap:silver",
      "urn:mace:incommon:iap:bronze"],
   "subject_types_supported":
     ["public", "pairwise"],
   "userinfo_signing_alg_values_supported":
     ["RS256", "ES256", "HS256"],
   "userinfo_encryption_alg_values_supported":
     ["RSA1_5", "A128KW"],
   "userinfo_encryption_enc_values_supported":
     ["A128CBC-HS256", "A128GCM"],
   "id_token_signing_alg_values_supported":
     ["RS256", "ES256", "HS256"],
   "id_token_encryption_alg_values_supported":
     ["RSA1_5", "A128KW"],
   "id_token_encryption_enc_values_supported":
     ["A128CBC-HS256", "A128GCM"],
   "request_object_signing_alg_values_supported":
     ["none", "RS256", "ES256"],
   "display_values_supported":
     ["page", "popup"],
   "claim_types_supported":
     ["normal", "distributed"],
   "claims_supported":
     ["sub", "iss", "auth_time", "acr",
      "name", "given_name", "family_name", "nickname",
      "profile", "picture", "website",
      "email", "email_verified", "locale", "zoneinfo",
      "http://example.info/claims/groups"],
   "claims_parameter_supported":
     true,
   "service_documentation":
     "http://server.example.com/connect/service_documentation.html",
   "ui_locales_supported":
     ["en-US", "en-GB", "en-CA", "fr-FR", "fr-CA"]
  }
```

Note:
* OIDC uses an endpoint on the provider so that clients can dynamically get the configuration, encryption keys and endpoint uris.

---

#### The cast

Provider

Client

Note:
* Outline who all the actors are in the flow with a name
* Go through each actor with setup

---

# Example

Note:
* Going to step through an example

---

### Step 1

```
HTTP/1.1 GET https://the.k8s.dashboard
```

```
302 Found
Location: https://<idp>/oauth2/auth?
  client_id=<kubernetes dashboard id>&
  redirect_uri=<url client can redirect back to>&
  scope=openid%20email%20profile
  ...
```

@[3]
@[4]
@[5]

---

### Step 2

```
HTTP/1.1 GET https://<idp>/oauth2/auth?
  client_id=<kubernetes dashboard id>&
  redirect_uri=<url client can redirect back to>&
  scope=openid%20email%20profile
  ...
```

<Idp starts login flow and we login>

```
302 Found
Location: https://<redirect uri from above>?code=<auth code>
```

---

### Step 3

```
HTTP/1.1 GET https://<redirect uri from above>?code=<auth code>
```

```
HTTP/1.1 POST https://<idp>/oauth2/token

{

}
```

```
200 Success

{

}
```

```
200 Success

<k8s dashboard html>
```

---

# Kubernetes Setup

Note:
* Lets go through how we actually set this up in Kubernetes for the api server
* There are several change points that need to be orchestrated together

---

#### Kubernetes API Server config

kube-apiserver cli flags

```
--oidc-ca-file # Can be left out to defer to host CAs
--oidc-client-id # Generated and given to you by IdP
--oidc-groups-claim # the claim key in the JWT for the groups
--oidc-groups-prefix
--oidc-issuer-url # The OIDC discovery endpoint
--oidc-username-claim # the claim key in the JWT for username
--oidc-username-prefix
```

@[5]
@[2]
@[6]
@[3]

Note:
* Show the api server params
* Explain what each one is for in the context of previous flow slides

---

#### Debugging and testing tokens

https://k8s.api.server/apis/authentication.k8s.io/v1beta1/tokenreviews

Request POST body:

```json
{
  "apiVersion": "authentication.k8s.io/v1beta1",
  "kind": "TokenReview",
  "spec": {
    "token": "<our JWT token we want to debug>"
  }
}
```

Response:

```json

```

Note:
* You MUST be authed and allowed to hit THIS endpoint

---

#### Openresty config

```
http {
  resolver 8.8.8.8;
  lua_package_path '~/lua/?.lua;;';
  # cache for JWT verification results
  lua_shared_dict introspection 10m;
  # cache for jwks metadata documents
  lua_shared_dict discovery 1m;
  server {
    listen 9000;
    set $session_secret os.getenv("SESSION_SECRET");
    set $session_cookie_secure on;
    large_client_header_buffers 4 32k;
    location / {
      access_by_lua '
        local opts = {
          client_id = os.getenv("OIDC_CLIENT_ID"),
          client_secret = os.getenv("OIDC_CLIENT_SECRET"),
          redirect_uri_path = "/oauth2/callback",
          discovery = "https://my.oauth2.server/.well-known/openid-configuration"
        }
        local res, err, _target, session = require("resty.openidc").authenticate(opts)
        if err or not res then
          ngx.status = 403
          ngx.say("forbidden")
          ngx.exit(ngx.HTTP_FORBIDDEN)
        end
        ngx.req.set_header("Authorization", "Bearer "..session.data.enc_id_token)
      ';
      proxy_pass http://localhost:9090;
    }
  }
}
```

@[16]
@[21-35]
@[22,25]
@[34]
@[36]

Using: [https://github.com/pingidentity/lua-resty-openidc](https://github.com/pingidentity/lua-resty-openidc)
