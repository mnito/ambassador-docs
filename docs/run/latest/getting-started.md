---
Title: Edge Stack quick start
description: "A simple three step guide to installing Edge Stack and quickly get started routing traffic from the edge of your Kubernetes cluster to your services."
---

import Alert from '@material-ui/lab/Alert';
import GettingStartedEdgeStack21Tabs from './gs-tabs'

# Edge Stack quick start

* [1. Installation](#1-installation)
* [2. Routing Traffic from the Edge](#2-routing-traffic-from-the-edge)
* [What's Next?]()

## 1. Installation

We'll start by installing Edge Stack into your cluster.

**We recommend using Helm** but there are other options below to choose from.

<GettingStartedEdgeStack21Tabs version="latest" />

<Alert severity="success"><b>Success!</b> At this point, you have installed Edge Stack. Now let's get some traffic flowing to your services.</Alert>

## 2. Routing traffic from the edge

Edge Stack uses Kubernetes Custom Resource Definitions (CRDs) to declaratively define its desired state. The workflow you are going to build uses a simple demo app, a **`Listener` CRD**, and a **`Mapping` CRD**. The `Listener` CRD tells Edge Stack what port to listen on, and the `Mapping` CRD tells Edge Stack how to route incoming requests by host and URL path from the edge of your cluster to Kubernetes services.

1. Start by creating a `Listener` resource for HTTP on port 8080:

```
kubectl apply -f - <<EOF
---
apiVersion: getambassador.io/v3alpha1
kind: Listener
metadata:
  name: $edgeStackName$-listener-8080
  namespace: $companyName$
spec:
  port: 8080
  protocol: HTTP
  securityModel: XFP
  hostBinding:
    namespace:
      from: ALL
---
apiVersion: getambassador.io/v3alpha1
kind: Listener
metadata:
  name: $edgeStackName$-listener-8443
  namespace: $companyName$
spec:
  port: 8443
  protocol: HTTPS
  securityModel: XFP
  hostBinding:
    namespace:
      from: ALL
EOF
```

2. Apply the YAML for the “Quote of the Moment" service.

  ```
  kubectl apply -f https://app.getambassador.io/yaml/v2-docs/latest/quickstart/qotm.yaml
  ```

  <Alert severity="info">The Service and Deployment are created in your default namespace. You can use <code>kubectl get services,deployments quote</code> to see their status.</Alert>

3. Apply the YAML for a `Mapping` to tell Edge Stack to route all traffic inbound to the `/backend/`
   path to the `quote` Service:

  ```yaml
  kubectl apply -f - <<EOF
  ---
  apiVersion: getambassador.io/v3alpha1
  kind: Mapping
  metadata:
    name: quote-backend
  spec:
    hostname: "*"
    prefix: /backend/
    service: quote
    docs:
      path: "/.ambassador-internal/openapi-docs"
  EOF
  ```

4. Store the Edge Stack load balancer IP address to a local environment variable. You will use this variable to test access to your service.

  ```
  export LB_ENDPOINT=$(kubectl -n $companyName$ get svc  $edgeStackName$ \
    -o "go-template={{range .status.loadBalancer.ingress}}{{or .ip .hostname}}{{end}}")
  ```

5. Test the configuration by accessing the service through the Edge Stack load balancer:

  ```
  $ curl -Lki https://$LB_ENDPOINT/backend/

    HTTP/1.1 200 OK
    content-type: application/json
    date: Wed, 23 Jun 2021 16:49:46 GMT
    content-length: 163
    x-envoy-upstream-service-time: 0
    server: envoy

    {
        "server": "serene-grapefruit-gjd4yodo",
        "quote": "The last sentence you read is often sensible nonsense.",
        "time": "2021-06-23T16:49:46.613322198Z"
    }
  ```

<Alert severity="success"><b>Victory!</b> You have created your first Edge Stack Mapping, routing a request from your cluster's edge to a service!</Alert>