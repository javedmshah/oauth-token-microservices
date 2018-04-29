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
