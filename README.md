## OPA-Envoy Decision Log Testing

This repository contains details of the test setup used to understand the impact of decision logging on the OPA-Envoy plugin.

### Platform

* Test Host: EC2 (_Instance type: m4.xlarge_)
* Minikube: `v1.26.0`
* Kubernetes: `v1.24.1`


### Setup

The setup comprises the OPA-Envoy plugin with DAS as the decision log server. To study the impact of decision logging
the following scenarios were tested:

* No decision logging
* HTTP decision logging
* HTTP and console decision logging

#### Envoy config

The Envoy configuration below defines an external authorization filter `envoy.ext_authz` for a gRPC authorization server.

```yaml
static_resources:
  listeners:
  - address:
      socket_address:
        address: 0.0.0.0
        port_value: 8000
    filter_chains:
    - filters:
      - name: envoy.filters.network.http_connection_manager
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
          codec_type: auto
          stat_prefix: ingress_http
          route_config:
            name: local_route
            virtual_hosts:
            - name: backend
              domains:
              - "*"
              routes:
              - match:
                  prefix: "/"
                route:
                  cluster: service
          http_filters:
          - name: envoy.ext_authz
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.http.ext_authz.v3.ExtAuthz
              transport_api_version: V3
              with_request_body:
                max_request_bytes: 8192
                allow_partial_message: true
              failure_mode_allow: false
              grpc_service:
                google_grpc:
                  target_uri: 127.0.0.1:9191
                  stat_prefix: ext_authz
                timeout: 0.5s
          - name: envoy.filters.http.router
  clusters:
  - name: service
    connect_timeout: 0.25s
    type: strict_dns
    lb_policy: round_robin
    load_assignment:
      cluster_name: service
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: 127.0.0.1
                port_value: 8080
admin:
  access_log_path: "/dev/null"
  address:
    socket_address:
      address: 0.0.0.0
      port_value: 8001
layered_runtime:
  layers:
    - name: static_layer_0
      static_layer:
        envoy:
          resource_limits:
            listener:
              example_listener_name:
                connection_limit: 10000
        overload:
          global_downstream_max_connections: 50000
```

Create the ConfigMap:

```bash
kubectl create configmap proxy-config --from-file envoy.yaml
```


### 3. OPA policy

Create an Envoy system type in DAS and download a policy bundle with the default policy:

```yaml
package policy.ingress

# Add policy/rules to allow or deny ingress traffic

allow = true
```

We will now serve this OPA bundle using Nginx.

```bash
docker run --rm --name bundle-server -d -p 8888:80 -v ${PWD}:/usr/share/nginx/html:ro nginx:latest
```

The above command will start a Nginx server running on port `8888` on your host and act as a bundle server.

> If using DAS as bundle server, it is not needed to download and locally host the bundle.

#### App Deployment with OPA and Envoy sidecars

Our deployment contains a sample Go app which provides information about
employees in a company. It exposes a `/people` endpoint to `get` and `create`
employees. More information can on the app be found
[here](https://github.com/ashutosh-narkar/go-test-server).

OPA is started with a configuration that sets the listening address of Envoy
External Authorization gRPC server and specifies the name of the policy decision
to query.

Save the deployment as **deployment.yaml**:

```yaml
kind: Deployment
apiVersion: apps/v1
metadata:
  name: example-app
  labels:
    app: example-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: example-app
  template:
    metadata:
      labels:
        app: example-app
    spec:
      initContainers:
        - name: proxy-init
          image: openpolicyagent/proxy_init:v5
          args: ["-p", "8000", "-u", "1111", "-w", "8282"]
          securityContext:
            capabilities:
              add:
              - NET_ADMIN
            runAsNonRoot: false
            runAsUser: 0
      containers:
      - name: app
        image: openpolicyagent/demo-test-server:v1
        ports:
        - containerPort: 8080
      - name: envoy
        image: envoyproxy/envoy:v1.20.0
        resources:
          limits:
            cpu: "1"
        volumeMounts:
        - readOnly: true
          mountPath: /config
          name: proxy-config
        args:
        - "envoy"
        - "--config-path"
        - "/config/envoy.yaml"
        env:
        - name: ENVOY_UID
          value: "1111"
      - name: opa
        image: openpolicyagent/opa:0.42.2-envoy
        resources:
          limits:
            cpu: "1"
        args:
        - "run"
        - "--server"
        - "--addr=localhost:8181"
        - "--diagnostic-addr=0.0.0.0:8282"
        - "--set=labels.system-id=<DAS_SYSTEM_ID>"
        - "--set=labels.system-type=template.envoy:2.1"
        - "--set=default_decision=main/main"
        - "--set=default_authorization_decision=/system/authz/allow"
        - "--set=services.default.url=http://host.minikube.internal:8888"
        - "--set=services.styra.url=https://TENANT.styra.com/v1"
        - "--set=services.styra.credentials.bearer.token=<DAS_ACCESS_TOKEN>"
        - "--set=bundles.default.resource=bundle.tar.gz"
        - "--set=plugins.envoy_ext_authz_grpc.addr=:9191"
        - "--set=plugins.envoy_ext_authz_grpc.path=main/main"
        - "--set=decision_logs.console=true"
        - "--set=decision_logs.reporting.min_delay_seconds=10"
        - "--set=decision_logs.reporting.max_delay_seconds=15"
        - "--set=decision_logs.reporting.upload_size_limit_bytes=134144"
        - "--set=decision_logs.service=styra"
        - "--ignore=.*"
        livenessProbe:
          httpGet:
            path: /health?plugins
            scheme: HTTP
            port: 8282
          initialDelaySeconds: 5
          periodSeconds: 5
        readinessProbe:
          httpGet:
            path: /health?plugins
            scheme: HTTP
            port: 8282
          initialDelaySeconds: 5
          periodSeconds: 5
      volumes:
      - name: proxy-config
        configMap:
          name: proxy-config
```

Replace `DAS_SYSTEM_ID`, `TENANT.styra.com` and `DAS_ACCESS_TOKEN` with the appropriate values.

The above manifest enables HTTP and console decision logging. Update the manifest as per the test scenario.

```bash
kubectl apply -f deployment.yaml
```

Check that the Pod shows `3/3` containers `READY` the `STATUS` as `Running`:

```bash
kubectl get pod

NAME                           READY   STATUS    RESTARTS   AGE
example-app-67c644b9cb-bbqgh   3/3     Running   0          8s
```

### Create a Service to expose HTTP server

In a second terminal, start a [minikube tunnel](https://minikube.sigs.k8s.io/docs/handbook/accessing/#using-minikube-tunnel) to allow for use of the `LoadBalancer` service type.

```bash
minikube tunnel
```

In the first terminal, create a `LoadBalancer` service for the deployment.

```bash
kubectl expose deployment example-app --type=LoadBalancer --name=example-app-service --port=8080
```

Check that the Service shows an `EXTERNAL-IP`:

```bash
kubectl get service example-app-service

NAME                  TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)          AGE
example-app-service   LoadBalancer   10.109.64.199   10.109.64.199   8080:32170/TCP   5s
```

Set the `SERVICE_URL` environment variable to the service's IP/port.

**minikube:**

```bash
export SERVICE_HOST=$(kubectl get service example-app-service -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
export SERVICE_URL=$SERVICE_HOST:8080
echo $SERVICE_URL
```

**minikube (example):**

```
10.109.64.199:8080
```

###  Run the test

Query the `http://$SERVICE_URL/people` endpoint to exercise the policy and calculate the end-to-end latency. [This](https://github.com/ashutosh-narkar/stress-opa-envoy)
sample script captures the end-to-end latency of an API request and then uses OPA's [metrics](https://pkg.go.dev/github.com/open-policy-agent/opa@v0.42.2/metrics)
package to generate a histogram of the latency measurements. The script makes API calls in a 10-second interval and prints
out the latency distribution.