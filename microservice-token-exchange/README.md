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
    "configOverridePath" : "&{EXCHANGE_POLICY_JSON_CONFIG_PATH|}",

    "resources" : [
        {
            "patterns" : [
                "http://resource1.example.com/*",
                "http://resource1.example.com:80/*",
                "https://resource1.example.com/*",
                "https://resource1.example.com:443/*"
            ],
            "audience" : "resource1",
            "scopes" : [
                "resource_read",
                "resource_write"
            ],
            "expirationSeconds" : 3210,
            "allowedActors" : [ "client3" ],
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
```text
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
```
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

# Token Exchange Policy - Open Policy Agent (OPA)

The token exchange microservice supports [Open Policy Agent](http://www.openpolicyagent.org/) (OPA) backed policy implementation. OPA policies provide logic that decides whether a token-exchange is allowed and what scopes and claims will be set into the generated token.

# Settings

This Token-Exchange Policy implementation matches requested resource `"patterns"` to a given `"audience"`,
which will be set into the generated token. Alternatively, if the token-exchange-request specifies an `"audience"` but
no resource URIs, then this policy will match that audience claim. Policy rules are _not_ mutually exclusive, in that
multiple policies could match for requested resources/audiences, and the result will be merged together for the
generated token. 

This is a powerful model for generating composite tokens only for specifically named audiences.
The JSON configuration file for this module is located at `conf/exchangepolicy_opa.json` and is shown below:

```json
{
    // Comma-separated names of Issuer JSON Web Key Store implementations, which maps OAuth token issuers to JSON Web Keys
    "issuerJsonWebKeyStore" : "&{ISSUER_JWK_STORE|json}",

    // Open Policy Agent URL for the token exchange policy
    "url":"&{EXCHANGE_POLICY_OPA_URL|http://localhost:8181/v0/data/forgerock/exchange/match}"
}
```

## Environment Variables

| Name                      | Description |
| ------------------------- | ----------- |
| `ISSUER_JWK_STORE`        | Comma-separated, priority ordered list of Issuer JSON Web Key Store implementations to use, which is `json` by default (See `conf/service.json`). |
| `EXCHANGE_POLICY_OPA_URL` | Open Policy Agent URL for the token exchange policy, which usually runs as a local service. |

# Open Policy Agent

The Open Source [Open Policy Agent](http://www.openpolicyagent.org/) project has been around since 2016 and has added many powerful features along the way. Examples include the _Rego_ scripting language, based on Datalog, and HTTP capabilities which make it possible to manage policies as part of your overall micro-gated microservice infrastructure. 

While this Token Exchange module is opinionated about the JSON payload that is sent to OPA and the JSON response from OPA, but there is great flexibility in how policy rules are
evaluated inside OPA itself.

An example HTTP request JSON payload, sent to OPA for evaluation, is:

```json
{
  "resources" : [
    "http://resource1.example.com/endpoint"
  ],
  "audiences" : [
    "resource2"
  ]
}
```

This example requests a resource name, which is associated with "resource1", and also explicitly requests access
to "resource2".

## exchangePolicy.json

Example `exchangePolicy.json` file, which defines the policy rule data inside OPA:

```json
{
    "resources" : [
        {
            "patterns" : [
                "^https?://resource1.example.com/.*",
                "^http://resource1.example.com:80/.*",
                "^https://resource1.example.com:443/.*"
            ],
            "audience" : "resource1",
            "scopes" : [
                "resource_read",
                "resource_write"
            ],
            "expirationSeconds" : 3210,
            "allowedActors" : [ "client3" ],
            "additionalClaims" : {
                "exampleString" : "value",
                "exampleNumber" : 1,
                "exampleBoolean" : true,
                "exampleObject" : { "key" : "value" },
                "exampleArray" : [ "item" ]
            }
        },
        {
            "patterns" : [
                "^http://resource2.example.com/.*",
                "^http://resource2.example.com:80/.*"
            ],
            "audience" : "resource2",
            "scopes" : [
                "resource_read"
            ]
        }
    ]
}
```

The JSON schema above is similar to the previously described JSON Token Exchange Policy module, which is the default exchange policy implementation. One notable difference is that the `"patterns"` in this case are full fledged regular expressions, instead of the simpler pattern language assumed by the other module. Although we describe URI patterns above, resources are _not_ restricted to be URIs. If at least one resource pattern matches, or the _audience_ matches, then then rule matches. 

A token exchange request may consist of requested resources, audiences, or both. This module considers it an error if neither resources or audiences are passed in the token exchange request.

These JSON definitions specify how resources and/or audiences are related within exchange policies, and also follow
the JSON schema expected by _this exchange policy module_ in JSON responses from OPA. The following fields are
expected to be returned by OPA:

- audience (required) : audience on the newly issued token
- scopes (optional) : scopes placed on the newly issued token
- expirationSeconds (optional) : must be 0 or higher, or uses the default in `conf/services.json`
- allowedActors (optional) : subjects of callers (bearer token) that can be exchange actors (all allowed by default)
- additionalClaims (optional) : which are ad-hoc claims added to the token and cannot override mandatory OAuth 2 claims

## exchangePolicy.rego

Example `exchangePolicy.rego` file, which defines the logic of the exchange policy is shown here. This example Rego file supports matching of items inside the `"resources"` array, shown above in the example `exchangePolicy.json` file, with requested audiences and/or resources present in the request. The response from the `match` rule returns an array of matching items from the `"resources"` array. Other than the restriction on the JSON request and JSON response formats, you can perform whatever complex evaluations are necessary in your own Rego files.

```text
package forgerock.exchange

import data.resources

##
# RUN IN SERVER MODE:
#
# ./opa run --server  exchangePolicy.json exchangePolicy.rego
#

lookupByResourcePattern = output {
  z = [r | re_match(resources[i].patterns[_], input.resources[_]); resources[i] = r]
  # convert array into set
  output = {x | x = z[_]}
}

lookupByAudience = output {
  z = [r | resources[i].audience = input.audiences[_]; resources[i] = r]
  # convert array into set
  output = {x | x = z[_]}
}

# curl -H "Content-Type: application/json" --request POST --data-binary "@exchangeInput.json" http://localhost:8181/v0/data/forgerock/exchange/match
match = output {
  a = lookupByAudience
  b = lookupByResourcePattern
  output = a | b
}
```

## Run OPA Server

Run the OPA server locally on the default port (8181):

```text
./opa run --server  exchangePolicy.json exchangePolicy.rego
```

## HTTP Request from Token Exchange Microservice

Example HTTP request payload from the Token Exchange Microservice, which you could save to `exchangeInput.json`:

```json
{
  "resources" : [
    "http://resource1.example.com/endpoint"
  ],
  "audiences" : [
    "resource2"
  ]
}
```

Issue the example HTTP request via:

```text
curl -H "Content-Type: application/json" \
    --request POST \
    --data-binary "@exchangeInput.json" \
    http://localhost:8181/v0/data/forgerock/exchange/match
```
