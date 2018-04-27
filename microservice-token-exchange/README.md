Deploy the token exchange container:
``kubectl create -f kube-exchange-env-var.yaml``

# Exchange tokens using a declarative JSON-based policy backend

The JSON file backed Token-Exchange Policy implementation provides logic that decides whether a token-exchange
is allowed and what scopes or claimsets will be returned in the generated access token.

This Token-Exchange Policy implementation matches requested resource URI `"patterns"` to a given `"audience"`,
which will appear in the generated token. Alternatively, if the token-exchange-request specifies an `"audience"` but
no resource URIs, then this policy will match that audience. Policy rules are _not_ mutually exclusive, in that
multiple policies could match for requested resources/audiences, and the result will be merged together for the
generated token.

The example JSON configuration can be overridden either by modifying `conf/exchangepolicy_json.json` or by
setting the optional `EXCHANGE_POLICY_JSON_CONFIG_PATH` environment variable, which is useful in a container
environment (e.g., Docker/Kubernetes).

```json
{
    // (optional) path of JSON config file that overrides this config file, and must only contain simple JSON syntax;
    // property-substitution values and/or inline comments are not allowed
    "configOverridePath" : "&{EXCHANGE_POLICY_JSON_CONFIG_PATH|}",

    "resources" : [
        {
            // patterns to match against the requested resource (either exact-match, or prefix-match up to first *)
            "patterns" : [
                "http://resource1.example.com/*",
                "http://resource1.example.com:80/*",
                "https://resource1.example.com/*",
                "https://resource1.example.com:443/*"
            ],
            // audience associated with this policy
            "audience" : "resource1",
            // scopes provided when resource-patterns and/or audience match
            "scopes" : [
                "resource_read",
                "resource_write"
            ],
            // optional field, to override number of seconds until token expiration, or null
            "expirationSeconds" : 3210,
            // optional array of actor_token subjects allowed to act as actors, or null
            "allowedActors" : [ "client3" ],
            // (optional) additional claims to be copied into generated tokens, or null
            "additionalClaims" : {
                "exampleString" : "value",
                "exampleNumber" : 1,
                "exampleBoolean" : true,
                "exampleObject" : { "key" : "value" },
                "exampleArray" : [ "item" ]
            }
        }
    ]
}
```
<!--
  ~ Copyright 2017 ForgeRock AS. All Rights Reserved
  ~
  ~ Use of this code requires a commercial software license with ForgeRock AS.
  ~ or with one of its affiliates. All use shall be exclusively subject
  ~ to such license between the licensee and ForgeRock AS.
  -->

# Exchange an access token in-place with a JWT - without requiring a PDP decision

This implementation will perform a simple token exchange, for a JWT, given any token that can be
introspected by the configured Token Introspection service.

Specification for this implementation:
µTE stands for Token Exchange microservice.

Case 1 : Stateful OAuth2 Token -> Stateless JWT
Flow: stateful AT to µTE-> µTE validates AT by proxy to AS->µTE mints new JWT (signs with own key).
a) Client 1 presents stateful AT to µTE service along with its own bearer token
b) µTE service validates AT (by proxy to Authz Server such as AM, of course) 
c) µTE service mints new stateless JWT using its own private key with contents of AT and returns stateless JWT.

Case 2 : Stateless OAuth2 Token -> Stateless JWT
Flow: stateless AT to µTE-> µTE validates AT->µTE mints new JWT (signs with own key).
a) Client 1 presents stateless AT to µTE service along with its own bearer token
b) µTE service validates AT after validating signature
c) µTE service mints new stateless JWT using its own private key with contents of AT and returns stateless JWT.
Note: The µTE will sign new JWT with its own key. Client 1 would receive the newly minted JWT, and eventually use it with Client 2 - and Client2 must call token validator (µTV) to validate (therefore µTV must be in possession of pub key as well).

There is no "resource" or "audience" validation requested along with a request to exchange an AM AT token with a stateless JWT. And there is no policy evaluation either.

Implementation and usage notes:

- The `Bearer` token, used to authenticate, must have scope claims for _both_ token exchange and token introspection
- Exchange request must have `requested_token_type=urn:ietf:params:oauth:token-type:jwt`
- The `audience` and `resource` parameters, in the exchange request, are currently _ignored_ and not applicable
- If the given `subject_token` can be introspected, then a JWT is returned containing the claims of the `subject_token`
- When the `subject_token` does _not_ contain the `sub`, `aud`, and `scope` or `scp` claims, they are given a `N_A` value
- If the exchange request specifies one or more optional `scope` parameters, those scopes must exist in the `subject_token`
- The optional exchange request `actor_token` must be a JWT signed by one of the keys in the configured Issuer JSON Web Key Store
- If an `actor_token` is specified, the optional `may_act` claim will be respected if found in the `subject_token`

## Settings

Environment variables are specified in the YAML. There is no need to set them explicitly in the json config.
The `conf/resources/exchangepolicy_jwt.json` configuration file requires an `"introspectUrl"`, which points to a
`microservice-token-validation` endpoint, and the `"issuerJsonWebKeyStore"` field, which is currently only used to
validate the optional `actor_token`'s signature.

```json
{
    // URL to Token Introspection service
    "introspectUrl" : "&{EXCHANGE_JWT_INTROSPECT_URL|http://localhost:8080/service/introspect}",

    // Comma-separated names of Issuer JSON Web Key Store implementations, which maps OAuth token issuers to JSON Web Keys
    // NOTE: this is currently only used to validate the optional actor_token
    "issuerJsonWebKeyStore" : "&{ISSUER_JWK_STORE|json}"
}
```

## Environment Variables

| Name                          | Description |
| ----------------------------- | ----------- |
| `EXCHANGE_JWT_INTROSPECT_URL` | URL to Token Introspection service (see `conf/exchangepolicy_jwt.json`) |
| `ISSUER_JWK_STORE`            | Comma-separated, priority ordered list of Issuer JSON Web Key Store implementations to use, which is `json` by default (See `conf/service.json`). |

## Example Usage

The environment variables are setup in the YAML file:
Use the POSTMAN collection to generate bearer and subject tokens.
Exchange `subject_token` for a JWT:

```text
curl -k \
	-d "grant_type=urn:ietf:params:oauth:grant-type:token-exchange" \
	-d "subject_token=$SUBJECT_TOKEN" \
	-d "subject_token_type=urn:ietf:params:oauth:token-type:access_token" \
	-d "requested_token_type=urn:ietf:params:oauth:token-type:jwt" \
    -H "Authorization: Bearer $AUTH_TOKEN" \
    -H "Content-Type: application/x-www-form-urlencoded" \
    -X POST "http://localhost:8080/service/tokensts" | jq
```

Exchange `subject_token`, with optional `actor_token` specified, for a JWT:

```text
curl -k \
	-d "grant_type=urn:ietf:params:oauth:grant-type:token-exchange" \
	-d "subject_token=$SUBJECT_TOKEN" \
	-d "subject_token_type=urn:ietf:params:oauth:token-type:access_token" \
	-d "actor_token=$AUTH_TOKEN" \
	-d "actor_token_type=urn:ietf:params:oauth:token-type:access_token" \
	-d "requested_token_type=urn:ietf:params:oauth:token-type:jwt" \
    -H "Authorization: Bearer $AUTH_TOKEN" \
    -H "Content-Type: application/x-www-form-urlencoded" \
    -X POST "http://localhost:8080/service/tokensts" | jq
```
# Exchange access tokens using OpenAM as a policy backend

Policy evaluation will result in response attributes which will be included as claims in the generated JWT access token.
Support for delegation using the may_act claim in the incoming `subject_token` is also provided. The `actor_token` and `actor_token_type` must be provided with a matching sub claim, and specific configurations in AM must be completed. These are already included in the amster config export included in the repository.
Details [here](https://forum.forgerock.com/2018/04/token-exchange-and-delegation/).

Delegation Support for actor definition.

The `subject_token` includes may_act as shown:
```json
{
    "at_hash": "EpGvTWe0WBJGJEdgUF2-2w",
    "sub": "Alice",
    "auditTrackingId": "079fda18-c96f-4c78-90f2-fc9be0545b32-204563",
    "iss": "http://localhost:8080/openam/oauth2",
    "tokenName": "id_token",
    "aud": "oidccient",
    "azp": "oidccient",
    "auth_time": 1523084106,
    "realm": "/",
    "may_act": {
        "sub": "Bob"
    },
    "exp": 1523087706,
    "tokenType": "JWTToken",
    "iat": 1523084106
}
```
The `actor_token` is:
```json
{
    "at_hash": "vMH7NUGN9cCADdI_Dp5bSg",
    "sub": "Bob",
    "auditTrackingId": "079fda18-c96f-4c78-90f2-fc9be0545b32-204537",
    "iss": "http://localhost:8080/openam/oauth2",
    "tokenName": "id_token",
    "aud": "oidcclient",
    "azp": "oidcclient",
    "auth_time": 1523084104,
    "realm": "/",
    "exp": 1523087704,
    "tokenType": "JWTToken",
    "iat": 1523084104
}
```


An example of using delegation is as follows:
curl -k     
    -d "grant_type=urn:ietf:params:oauth:grant-type:token-exchange" \
    -d "resource=http://images.example.com:*/*" \
    -d "subject_token=$EX_SUBJECT_TOKEN" \
    -d "subject_token_type=urn:ietf:params:oauth:token-type:id_token" \
    -d "actor_token=$EX_ACTOR_TOKEN" \
    -d "actor_token_type=urn:ietf:params:oauth:token-type:id_token" \
    -H "Authorization: Bearer $EX_BEARER_TOKEN" \
    -H "Content-Type: application/x-www-form-urlencoded" \
    -X POST "http://localhost:38080/service/tokensts" | jq

If the AM policy check is successful, the returned JWT accesst_token would be:

```json
{
    "sub": "user.0",
    "aud": "example.com",
    "scp": [
        "read",
        "write"
    ],
    "act": {
        "sub": "user.1"
    },
    "iss": "https://fr.tokenexchange.example.com",
    "exp": 1523382895,
    "iat": 1523379295
}
```
