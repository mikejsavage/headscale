# Running headscale behind a reverse proxy

Running headscale behind a reverse proxy is useful when running multiple applications on the same server, and you want to reuse the same external IP and port - usually tcp/443 for HTTPS.

### WebSockets

The reverse proxy MUST be configured to support WebSockets, as it is needed for clients running Tailscale v1.30+.

WebSockets support is required when using the headscale embedded DERP server. In this case, you will also need to expose the UDP port used for STUN (by default, udp/3478). Please check our [config-example.yaml](https://github.com/juanfont/headscale/blob/main/config-example.yaml).

### TLS

Headscale can be configured not to use TLS, leaving it to the reverse proxy to handle. Add the following configuration values to your headscale config file.

```yaml
server_url: https://<YOUR_SERVER_NAME> # This should be the FQDN at which headscale will be served
listen_addr: 0.0.0.0:8080
metrics_listen_addr: 0.0.0.0:9090
tls_cert_path: ""
tls_key_path: ""
```

## nginx

The following example configuration can be used in your nginx setup, substituting values as necessary. `<IP:PORT>` should be the IP address and port where headscale is running. In most cases, this will be `http://localhost:8080`.

```Nginx
map $http_upgrade $connection_upgrade {
    default      keep-alive;
    'websocket'  upgrade;
    ''           close;
}

server {
    listen 80;
	listen [::]:80;

	listen 443      ssl http2;
	listen [::]:443 ssl http2;

    server_name <YOUR_SERVER_NAME>;

    ssl_certificate <PATH_TO_CERT>;
    ssl_certificate_key <PATH_CERT_KEY>;
    ssl_protocols TLSv1.2 TLSv1.3;

    location / {
        proxy_pass http://<IP:PORT>;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        proxy_set_header Host $server_name;
        proxy_redirect http:// https://;
        proxy_buffering off;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $http_x_forwarded_proto;
        add_header Strict-Transport-Security "max-age=15552000; includeSubDomains" always;
    }
}
```

## istio/envoy

If you using [Istio](https://istio.io/) ingressgateway or [Envoy](https://www.envoyproxy.io/) as reverse proxy, there are some tips for you. If not set, you may see some debug log in proxy as below:

```log
Sending local reply with details upgrade_failed
```

### Envoy

You need add a new upgrade_type named `tailscale-control-protocol`. [see detail](https://www.envoyproxy.io/docs/envoy/latest/api-v3/extensions/filters/network/http_connection_manager/v3/http_connection_manager.proto#extensions-filters-network-http-connection-manager-v3-httpconnectionmanager-upgradeconfig)

### Istio

Same as envoy, we can use `EnvoyFilter` to add upgrade_type.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: headscale-behind-istio-ingress
  namespace: istio-system
spec:
  configPatches:
    - applyTo: NETWORK_FILTER
      match:
        listener:
          filterChain:
            filter:
              name: envoy.filters.network.http_connection_manager
      patch:
        operation: MERGE
        value:
          typed_config:
            "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
            upgrade_configs:
              - upgrade_type: tailscale-control-protocol
```

## Caddy

The following Caddyfile is all that is necessary to use Caddy as a reverse proxy for headscale, in combination with the `config.yaml` specifications above to disable headscale's built in TLS. Replace values as necessary - `<YOUR_SERVER_NAME>` should be the FQDN at which headscale will be served, and `<IP:PORT>` should be the IP address and port where headscale is running. In most cases, this will be `localhost:8080`.

```
<YOUR_SERVER_NAME> {
    reverse_proxy <IP:PORT>
}
```

Caddy v2 will [automatically](https://caddyserver.com/docs/automatic-https) provision a certficate for your domain/subdomain, force HTTPS, and proxy websockets - no further configuration is necessary.

For a slightly more complex configuration which utilizes Docker containers to manage Caddy, Headscale, and Headscale-UI, [Guru Computing's guide](https://blog.gurucomputing.com.au/smart-vpns-with-headscale/) is an excellent reference.

## Apache

The following minimal Apache config will proxy traffic to the Headscale instance on `<IP:PORT>`. Note that `upgrade=any` is required as a parameter for `ProxyPass` so that WebSockets traffic whose `Upgrade` header value is not equal to `WebSocket` (i. e. Tailscale Control Protocol) is forwarded correctly. See the [Apache docs](https://httpd.apache.org/docs/2.4/mod/mod_proxy_wstunnel.html) for more information on this.

```
<VirtualHost *:443>
	ServerName <YOUR_SERVER_NAME>

	ProxyPreserveHost On
	ProxyPass / http://<IP:PORT>/ upgrade=any

	SSLEngine On
	SSLCertificateFile <PATH_TO_CERT>
	SSLCertificateKeyFile <PATH_CERT_KEY>
</VirtualHost>
```
