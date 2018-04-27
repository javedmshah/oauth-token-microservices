# Quickstart
Deploy the Token Validation microservice container :
```shell
kubectl create -f kube-validation-env-var.yaml
```
The environment variables are already set in YAML file. There is no need to provide them explicitly.

The Token Validation Microservice is a Java application (JDK 1.7 or higher) that introspects and validates OAuth 2.0 tokens.

# Example REST Endpoints
To access the "ping" endpoint with the "info" user-account. 
Here we use the minikube ip and the netPort defined in the YAML:

curl -k -u info:info --request GET --url 'http://192.68.99.100:30103/validation/info/ping'
To exercise the token introspection endpoint:

The attached POSTMAN collection in this repository provides examples to generate the BEARER and other tokens described here.

Generate a token to use as an Authorization Bearer token. Use example "client3" account that has "introspect" scope. Also use the "client1" account to generate a token to introspect. 

> microservice-token-validation
curl -k \
    -d "token_type_hint=access_token&token=$INTROSPECT_TOKEN" \
    -H "Authorization: Bearer $BEARER_TOKEN" \
    -H "Content-Type: application/x-www-form-urlencoded" \
    -X POST "http://192.68.99.100:30103/service/introspect" | jq

Settings
Token Signature Verification
The "issuerJsonWebKeyStore" field in token-validation-zip/src/main/resources/conf/service.json allows you to configure which JSON Web Key (JWK) settings to use when verifying token signatures for specific OAuth 2.0 Token Issuers. In the repository example, the default Issuer JSON Web Key Store is the json implementation, which is defined by a JSON configuration file. 
```
{
    // Comma-separated names of Issuer JSON Web Key Store implementations, which maps OAuth token issuers to JSON Web Keys
    "issuerJsonWebKeyStore" : "&{ISSUER_JWK_STORE|json}",

    // Comma separated list of Token-Introspection API implementations (e.g., "json,openam")
    "introspectionServices" : "&{INTROSPECTION_SERVICES|json}",

    // scope required, on bearer token, to perform token introspection
    "requiredBearerScope" : "&{INTROSPECTION_REQUIRED_SCOPE|introspect}"
}
```
Likewise, the "introspectionServices" field configures one or more Token Introspection API implementations by setting the value to a comma-separated list of implementation names. Each implementation is given a chance to handle the token, in given order, until an implementation successfully handles the token. An error response occurs if no implementation successfyly introspects a given token.

The following implementations are provided:
```
json : Configured by a JSON file (See the token-validation-introspect-json module README.md file.)
openam : Proxies introspect calls to OpenAM (See the token-validation-introspect-openam module README.md file.)
```
Environment Variables
The application is configured with example settings by default. Oftentimes you will need to provide values for the following environment variables for more complex demonstrations or for production use.

> Incoming Transaction ID
Each incoming HTTP request is assigned a transaction ID, for audit logging purposes, which by default is a UUID. ForgeRock Microservices, and other products (e.g., OpenAM), support propagation of a transaction ID between point-to-point service calls. To enable reading a transaction ID from the X-ForgeRock-TransactionId HTTP header, you must enable the following system property (defined in conf/system.properties):

`org.forgerock.http.TrustTransactionHeader=true`
To enable this feature, set the TRUST_TRANSACTION_ID_HEADER environment variable in the YAML file if needed:

`export TRUST_TRANSACTION_ID_HEADER=true`

## Prometheus
We provide a Prometheus endpoint which is convenient for monitoring published metrics. Additional details and alternative metrics formats are described in the core-metrics module of the microservice-core project.

Below is an example prometheus.yml file, from the token-validation-samples/metrics/ directory, for connecting to the metrics endpoint. In production, you should configure the connection to use HTTPS. The following example uses insecure HTTP here for simplicity:

global:
  scrape_interval: 15s
  external_labels:
    monitor: 'my_prometheus'

>https://prometheus.io/docs/operating/configuration/#scrape_config
scrape_configs:
  - job_name: 'microservice-token-validation'
    scrape_interval: 5s
    metrics_path: '/validation/metrics/prometheus'
    scheme: http
    params:
      _fields: ['text']
      _mimeType: ['text/plain;version=0.0.4']
    basic_auth:
      username: 'metrics'
      password: 'metrics'
    static_configs:
      - targets: ['localhost:8080']
To install and run on OSX, via Homebrew:

brew update
brew install prometheus

prometheus -config.file=token-validation-samples/metrics/prometheus.yml
Test the connection via the basic Prometheus website at http://localhost:9090/.

## Grafana
Once you are collecting metrics with Prometheus, you can generate graphs with Grafana.

To install Grafana via Homebrew:

brew update
brew install grafana
```
grafana-server --config=/usr/local/etc/grafana/grafana.ini \
  --homepath /usr/local/share/grafana \
  cfg:default.paths.logs=/usr/local/var/log/grafana \
  cfg:default.paths.data=/usr/local/var/lib/grafana \
  cfg:default.paths.plugins=/usr/local/var/lib/grafana/plugins
```
You can then access the Grafana website at http://localhost:3000/, add Prometheus as a datasource, and use the following example queries to build your graphs:

`rate(introspectRequests_total{job="microservice-token-validation"}[1m])` will graph Introspect Requests per Second
