# IntelliMon API Guard

This software acts as an API reverse-proxy/gateway for a Zabbix web frontend.

The main purpose is to allow public access to the Zabbix API without exposing the entire web frontend.

API access is controlled with bearer tokens, which are separate from the Zabbix API tokens.

## Target config and token-based access

The proxy only allows HTTP(S) requests that have the correct bearer token set via the `Authorization: Bearer ...` header. Since the request destination is determined by the token in the configuration, the request path is ignored. Therefore, it can be just `/` or the original path of the Zabbix frontend for drop-in replacements.  The proxy then takes any API requests that the Zabbix web frontend would normally receive and forwards both the request and the respose as-is.

Targets are defined in a YAML file that is passed as parameter `--targets`. Here is an example for a single Zabbix target:

```yaml
# Example of a target config for one Zabbix target
targets:
  zabbix:
    # the URL to your Zabbix frontend API, including "api_jsonrpc.php"
    url: http://zabbix.loc/api_jsonrpc.php
    # optional - path to a CA PEM file that was used to sign the Zabbix frontend's certificate
    # caPath: "/path/to/zabbix/ca.pem"
    # optional - set this to true to ignore certificate errors on the Zabbix frontend entirely
    # skipVerify: true
    tokens:
      # List of tokens assigned to this Zabbix server. API clients must include one of these
      # in the HTTP header as "Authorization: Bearer ...".
      # Each token can be any string, e.g. hex strings, base64 or just some text.
      - 0D6640E54634758727B0DD4C9D9F6009
      - EjNcAhScADha1pbkNYyzVg==
      - mytoken
```

It is also possible to define multiple Zabbix targets. The token decides what Zabbix target should receive the request. As a result, it's important that each token is uniquely assigned to one target only. The proxy will treat duplicate tokens as errors.

Here is an example for a multi-target config:

```yaml
# Example of a target config for multiple Zabbix targets
targets:
  # development server
  zabbixDev:
    # the URL to your Zabbix frontend API, including "api_jsonrpc.php"
    url: https://zabbix-dev.loc:8443/api_jsonrpc.php
    # optional - path to a CA PEM file that was used to sign the Zabbix frontend's certificate
    # caPath: "/path/to/zabbix/ca.pem"
    # optional - set this to true to ignore certificate errors on the Zabbix frontend entirely
    # skipVerify: true
    tokens:
      # List of tokens assigned to this Zabbix server. API clients must include one of these
      # in the HTTP header as "Authorization: Bearer ...".
      # Each token can be any string, e.g. hex strings, base64 or just some text.
      #
      # Note: Tokens must be unique and can't be reused between servers!
      #
      # Example for 128 bit hex tokens.
      #
      # Generate more with this command:
      # hexdump -vn16 -e'4/4 "%08X" 1 "\n"' /dev/urandom
      - 0D6640E54634758727B0DD4C9D9F6009
      - 79A17B763BF5DEDD95335E43AE6D6D69
      - 900963542AC3BA0F3E9190D8CC8B8E24

  # production server
  zabbixProd:
    # the URL to your Zabbix frontend API, including "api_jsonrpc.php"
    url: https://zabbix-prod.loc/api_jsonrpc.php
    # optional - path to a CA PEM file that was used to sign the Zabbix frontend's certificate
    caPath: "/etc/ssl/prod_ca.crt"
    # optional - set this to true to ignore certificate errors on the Zabbix frontend entirely
    # skipVerify: true
    tokens:
      # List of tokens assigned to this Zabbix server. API clients must include one of these
      # in the HTTP header as "Authorization: Bearer ...".
      # Each token can be any string, e.g. hex strings, base64 or just some text.
      #
      # Note: Tokens must be unique and can't be reused between servers!
      #
      # Example for 512 bit tokens encoded with base64.
      #
      # Generate more with this command:
      # dd if=/dev/urandom bs=64 count=1 2> /dev/zero | base64 -w0; echo
      - pCusUADrlVQhsteSRI0GeslEmNtr7VtErTHAk5ZshL8LqeeCQOt+Ethq1aTDHEe50xjAHrNTKmLpNKwd9Zg6GQ==
      - nCLfTzZaFcxnvgsxOVBI/GqMk9aXzyu5TsAKe1s/xlJ8JT+i4Mq3TLuZV1A9u52JzUuACr75DZoblu9U+b0krg==
      - LIHWjjBy/SiN3UTZz7en+RiVyOHlOw4Av29egn9nlR8ZnTsIC0ZQ1evGVy0/4JeEk69cyWNkvRmzjKqc3xIoWw==

```

The target config file can be edited while the proxy is running and is reloaded automatically. If an error occurs while reloading the config file, the last valid config will be used instead.

It is also allowed to start the proxy with a missing target config file. Without targets or tokens, any access to the proxy results in a `401 Unauthorized` respose, though.

## Command line example

This example starts an API reverse-proxy using targets from the target config file `targets.yaml` using the default port 8080:

```bash
./intmon-api-guard start --targets targets.yaml --listen ":8080"
```

To enable TLS, use `--tls` and then set the certificate path with `--tls-cert-path` and the key file with `--tls-key-path`:


```bash
./intmon-api-guard start --targets targets.yaml --tls --tls-cert-path server.crt --tls-key-path server.key --listen ":443"
```

Instead of command line parameters, a YAML config file can be used as well. In this case, set `--config` to the path of the config file:

```bash
./intmon-api-guard start --config intmon-api-guard.yaml
```

The content of `intmon-api-guard.yaml` can look like this:
```yaml
log-level: 6
targets: targets.yaml
tls: true
tls-cert-path: server.crt
tls-key-path: server.key 
listen: ":443"
```

If `--config` is not specified, `~/.intmon-api-guard.yaml` is checked and loaded if found.

The command line help can be retrieved using `start --help`:

```
Usage:
  intmon-api-guard start [flags]

Flags:
  -h, --help                   help for start
      --listen string          The address and port that the proxy should listen to (default ":8080")
      --targets string         Path to a target configuration file (default "targets.yaml")
      --tls                    Start proxy with TLS support
      --tls-cert-path string   Path to the proxy certificate file used for TLS (default "server.crt")
      --tls-key-path string    Path to the proxy certificate key file used for TLS (default "server.key")

Global Flags:
      --config string   config file (default is $HOME/.intmon-api-guard.yaml)
      --log-level int   Logging level between 0 and 6, higher values increase verbosity (default 4)
```

## Metrics

A Prometheus metrics endpoint is available at the path ``/metric``. Metrics are disabled on default and can be enabled in the target config:

```yaml
metrics:
  enabled: true
  tokens:
     - mytoken
     - mystatstoken
```

The list of tokens can be any token used in targets or new tokens used specifically for the metrics endpoint.

A Zabbix template is available in the folder [zabbix](./zabbix) that uses the metrics endpoint for monitoring.

After importing the template, assign it to a host of your choice and set ``{$METRIC_API_URL}`` to the proxy URL (with the path `/metric`) and ``{$METRIC_API_TOKEN}`` to a token that is listed in the metrics token list.

## Installation

### Docker

An example setup for Docker compose can be found in the folder [docker](./docker). Just add the binary of you choice to `docker/build` as `intmon_api_guard` (don't forget chmod 0755), then run `docker-compose up -d`.

## Testing the proxy

Assuming you have the single target configuration above, you can test the proxy on the same machine by using the CURL command below:

```bash
curl -X POST http://127.0.0.1:8080 -H 'Content-Type: application/json' -H 'Authorization: Bearer mytoken' -d '{"jsonrpc":"2.0","method":"apiinfo.version","params":{},"id":1}'
```

The proxy should then return the API version of the frontend server:

```
{"jsonrpc":"2.0","result":"5.0.26","id":1}
```

## HTTP codes

HTTP requests to the proxy may return the following status codes:

| HTTP Code              | Meaning                                                                   |
| ---------------------- | ------------------------------------------------------------------------- |
| 200 OK                 | Request was sent successfully to the Zabbix frontend.                     |
| 401 Unauthorized       | The bearer token is either missing or invalid.                            |
| 405 Method Not Allowed | Request was not sent as HTTP POST.                                        |
| 400 Bad Request        | Request was not sent with content type "application/json".                |
| 502 Bad Gateway        | Request could not be sent to the Zabbix frontend due to network issues.   |

The status codes and body from the Zabbix frontend are forwarded as is. In case there's no reply available, the body is left empty.
