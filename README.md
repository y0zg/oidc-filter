# build

```
export PATH=/home/user/.cargo/bin:$PATH
rustup update
rustup target add wasm32-unknown-unknown --toolchain nightly
#rustup target add wasm32-unknown-unknown
rustup +nightly --target wasm32-unknown-unknown
rustup run nightly cargo install cargo-modules
rustup run nightly cargo build --target wasm32-unknown-unknown

cargo  +nightly build --target wasm32-unknown-unknown --release

# clear 
# rm -rf  ~/.cargo/registry
```

# TLS

```
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: oidc-filter
spec:
  configPatches:
  - applyTo: HTTP_FILTER
    match:
      context: SIDECAR_INBOUND
      proxy:
        proxyVersion: '^1\.5.*'
      listener:
        filterChain:
          filter:
            name: envoy.http_connection_manager
            subFilter:
              name: envoy.filters.http.jwt_authn
    patch:
      operation: INSERT_BEFORE
      value:
        config:
          config:
            name: oidc-filter
            rootId: oidc-filter_root
            configuration: |
                {
                  "auth_cluster": "outbound|80||keycloak-http.keycloak.svc.cluster.local",
                  "auth_host": "keycloak-http.keycloak.svc.cluster.local:80",
                  "login_uri": "https://keycloak.example.com/auth/realms/istio/protocol/openid-connect/auth",
                  "token_uri": "https://keycloak.example.com/auth/realms/istio/protocol/openid-connect/token",
                  "client_id": "test",
                  "client_secret": "xxx"
                }
            vmConfig:
              code:
                local:
                  filename: /var/local/lib/wasm-filters/oidc.wasm
              runtime: envoy.wasm.runtime.v8
              vmId: oidc-filter
              allow_precompiled: true
        name: envoy.filters.http.wasm
  workloadSelector:
    labels:
      app: httpbin
```

```
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: httpbin
spec:
  gateways:
  - gateway
  hosts:
  - nginx.example.com
  http:
  - match:
    - uri:
        prefix: /
    route:
    - destination:
        host: httpbin.default.svc.cluster.local
        port:
          number: 8000

```
# oidc-filter



`oidc-filter` is a WASM plugin for Envoy/Istio that will redirect users to a given authentication URI if they do not present a JWT token.

## Features

- Automatically redirect users with no active session to an OpenID Connect Authorization Server for authorization
- Stores JWT in cookie and transparently writes it to `Authorization` header for every request

## How do I use this thing?

Check out the [example/](https://github.com/dgn/oidc-filter/tree/master/example/) directory.

## Limitations

- oidc-filter doesn't verify the JWTs yet (but Istio does that)
- If the token has expired, AJAX calls with methods other than GET will fail on first attempt (but then succeed afterwards)
- Not using state or nonce yet (so susceptible to replay attacks)

## Development

- Running `make build` in the root of the repository will build `oidc.wasm`
- See the [example/](https://github.com/dgn/oidc-filter/tree/master/example/) directory for how to test your changes

## TODO
- Add option to replay POST requests after redirects (so that redirected AJAX calls don't fail)
  - Not sure if that's good behaviour
