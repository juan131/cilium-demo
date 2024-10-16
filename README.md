# Cilium demno

Get started with Cilium and L7 Network Policies.

## Table of Contents

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Cilium demno](#cilium-demno)
  - [Table of Contents](#table-of-contents)
  - [Prerequisites](#prerequisites)
    - [Setup](#setup)
  - [Cilium Components](#cilium-components)
  - [Alba House example](#alba-house-example)
  - [Observability](#observability)
  - [Metrics](#metrics)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Prerequisites

The following tools are required to use this project:

- [Git](https://git-scm.com)
- [Kubernetes cluster](https://kubernetes.io/docs/setup)
- [Helm](https://helm.sh)
- [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets) & [Sealed Secrets Updater](https://github.com/juan131/sealed-secrets-updater)
- [git-crypt](https://github.com/AGWA/git-crypt)
- [yq](https://github.com/mikefarah/yq)

Please refer to each tool's installation instructions to pre-setup your environment.

### Setup

This repo follow's the repo structure from [k8s-gitops-template](https://github.com/juan131/k8s-gitops-template), refer to the [Initial Setup](https://github.com/juan131/k8s-gitops-template/blob/main/docs/tutorials/initial-setup.md) guide to setup a GKE cluster and the rest of CI/CD pipeline.

## Cilium Components

The following Cilium components are deployed in the cluster in the `kube-system` namespace:

- Cilium Agent (DaemonSet) compounded by the following sub-components:
  - Cilium Agent: listens for Kubernetes events to learn when containers or workloads are started and stopped. It manages the eBPF programs which the Linux kernel uses to control all network access in / out of those containers.
  - Hubble Server: (embedded into the Cilium agent) retrieves the eBPF-based visibility from Cilium.
  - Cilium Envoy: Sidecar container which is used by Cilium and its host proxy for enforcing HTTP and other L7 policies as specified in network policies for the cluster.
- Cilium Operator: manages duties in the cluster which should logically be handled once for the entire cluster, rather than once for each node in the cluster
- CNI Plugin: invoked by K8s when a pod is scheduled or terminated on a node. It interacts with the Cilium API of the node to trigger the necessary datapath configuration to provide networking, load-balancing and network policies for the pod.
- Hubble Relay: is aware of all running Hubble servers and offers cluster-wide visibility by connecting to their respective gRPC APIs and providing an API that represents all servers in the cluster
- Hubble UI: utilizes relay-based visibility to provide a graphical UI.

## Alba House example

This is an illustrative example of to understand the power of Cilium Network Policies and how they can be used to secure your applications. In this example we have:

- A "security system" (implemented as a REST API server) that provides access to different zones of Duenas' palace.
- The "duchess", (implemented as a REST client) who can access every zone in the palace, but can only access the "strongbox" when carrying the "key".
- The "gardener", (implemented as a REST client) who can exclusively access the "garden".
- The "thief", (implemented as a REST client) who cannot access any zone in the palace.

The security system is a dumb system developed by the Mossos d'Esquadra in charge of arresting Puidgemont, and we've recently discovered that it grants access to everyone, so we've decided to secure it with Cilium Network Policies.

Given the urgency of the situation, 1st we implemented an L3/L4 network policy to block all traffic to the security system unless it comes from "Alba House" employees. To do so:

- Apply the [L3/L4 network policy](./networkpolicies/l3-l4-policy.yaml).
- Test the network policy by trying to access the security system from the "thief" pod.

```bash
$ kubectl exec tejado -- curl -sX GET http://security/v1/mock/house
(...)
$ kubectl exec pepe -- curl -sX GET http://security/v1/mock/house
{"access":"granted"}
$ kubectl exec mrs-fitz-james-stuart-de-silva -- curl -sX GET http://security/v1/mock/house
{"access":"granted"}
```

> Note: It's expected that the blocked requests hang until they time out given the network policy simply drops the packets.

Once we addressed the most urgent issue, we realized the "gardener" was still able to access the "house" and "strongbox" zones. To fix this, we implemented L7 network policies to:

- Restrict access to the "house" zone to the "duchess".
- Restrict access to the "strongbox" zone to the "duchess" pod only when carrying the "key".

To do so:

- Apply the [L7 network policy](./networkpolicies/l7-policy.yaml).
- Test the network policy by trying to access the "house" and "strongbox" zones from the "gardener" pod.

```console
$ kubectl exec pepe -- curl -sX GET http://security/v1/mock/garden
{"access":"granted"}
$ kubectl exec pepe -- curl -sX GET http://security/v1/mock/house
Access denied
$ kubectl exec pepe -- curl -sX GET http://security/v1/mock/strongbox
Access denied
```

> Note: now an HTTP 403 access denied is sent back for HTTP requests which violate the policy instead of hanging.

- Test the network policy by trying to access the "house" and "strongbox" zones from the "duchess" pod.

```console
$ kubectl exec mrs-fitz-james-stuart-de-silva -- curl -sX GET http://security/v1/mock/house
{"access":"granted"}
$ kubectl exec mrs-fitz-james-stuart-de-silva -- curl -sX GET http://security/v1/mock/strongbox
Access denied
$ kubectl exec mrs-fitz-james-stuart-de-silva -- curl -sX GET -H 'X-API-KEY: vivaersevilla' http://security/v1/mock/strongbox
Access denied
$ kubectl exec mrs-fitz-james-stuart-de-silva -- curl -sX GET -H 'X-API-KEY: vivaerbetis' http://security/v1/mock/strongbox
{"access":"granted"}
```

## Observability

We can use the Hubble UI to visualize the network traffic in the cluster. To access the Hubble UI:

- Port-forward the Hubble UI service to your local machine:

```bash
kubectl port-forward svc/cilium-hubble-ui 8080:80 -n kube-system
```

- Open your browser and navigate to [http://localhost:8080](http://localhost:8080).

It's also possible to inspect the traffic using the [Hubble CLI](https://docs.cilium.io/en/stable/observability/hubble/setup/#hubble-cli-install). To do so:

- Download the TLS client certificate and key to your local machine:

```bash
kubectl get secret -n kube-system cilium-hubble-relay-client-crt -o json | jq -r '.data["tls.crt"]' | base64 --decode > tls.crt
kubectl get secret -n kube-system cilium-hubble-relay-client-crt -o json | jq -r '.data["tls.key"]' | base64 --decode > tls.key
```

- Port-forward the Hubble Relay service to your local machine:

```bash
kubectl port-forward svc/cilium-hubble-relay 4245:4245 -n kube-system
```

- Use the Hubble CLI to inspect the traffic:

```bash
hubble observe --tls --tls-allow-insecure --tls-client-cert-file tls.crt --tls-client-key-file tls.key
```

## Metrics

Cilium exposes metrics in Prometheus format which can be scraped by Prometheus. This repo contains the required manifests to setup kube-prometheus stack and Grafana Operator to visualize the metrics. These manifests include the ServiceMonitor objects and GrafanaDatasource objects to automatically discover and scrape the Cilium metrics.
You can also find a [Grafana dashboard](./dashboards/cilium.json) to visualize the Cilium metrics.

To access the Grafana UI:

- Obtain the Grafana admin password:

```bash
kubectl get secret -n monitoring grafana-operator-grafana-admin-credentials -o json | jq -r .data.GF_SECURITY_ADMIN_PASSWORD | base64 --decode
```

- Port-forward the Grafana service to your local machine:

```bash
kubectl port-forward svc/grafana-operator-grafana-service 3000:3000 -n monitoring
```

- Open your browser and navigate to [http://localhost:3000](http://localhost:3000) and login with the admin user and password obtained in the previous step.
