---
parser: v2
auto_validation: true
time: 30
tags: [ tutorial>intermediate, topic>cloud, software-product>sap-business-technology-platform]
primary_tag: software-product>sap-btp--kyma-runtime
---

# Send Requests Using Istio Egress Gateway in SAP BTP, Kyma Runtime
<!-- description --> Configure and use the Istio egress Gateway to allow outbound traffic from your Kyma runtime cluster to specific external destinations.

## Prerequisites
  - [The Istio module added](https://help.sap.com/docs/btp/sap-business-technology-platform/enable-and-disable-kyma-module?locale=en-US&version=Cloud) 
  - [kubectl configured to kubeconfig downloaded from SAP BTP, Kyma runtime](cp-kyma-download-cli)

## You will learn
  - How to configure the Istio egress Gateway to allow outbound traffic
  - How to send an HTTPS request to an external website

## Intro
Learn how to configure and use the Istio egress Gateway to allow outbound traffic from your Kyma runtime cluster to specific external destinations. Test your configuration by sending an HTTPS request to an external website using a sample Deployment.

---

### Configure the Istio egress Gateway

1. Enable the egress Gateway in the Istio custom resource:
    
    ```Shell/Bash
    kubectl apply -f - <<EOF
    apiVersion: operator.kyma-project.io/v1alpha2
    kind: Istio
    metadata:
      name: default
      namespace: kyma-system
      labels:
        app.kubernetes.io/name: default
    spec:
      components:
        egressGateway:
          enabled: true
    EOF
    ```
2. Enable additional sidecar logs to see the egress Gateway being used in requests:
    ```Shell/Bash
    kubectl apply -f - <<EOF
    apiVersion: telemetry.istio.io/v1
    kind: Telemetry
    metadata:
      name: mesh-default
      namespace: istio-system
    spec:
      accessLogging:
        - providers:
          - name: envoy
    EOF
    ```

### Create a sample Deployment

1. Export the name of the namespace in which you want to create a sample Deployment:
   
    ```Shell/Bash
    export NAMESPACE={service-namespace}
    ```

2. Create a new namespace for the sample application:

    ```Shell/Bash
    kubectl create ns $NAMESPACE
    kubectl label namespace $NAMESPACE istio-injection=enabled --overwrite
    ```
3. Apply the `curl` Deployment to send the requests:

    ```Shell/Bash
    kubectl apply -f - <<EOF
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: curl
      namespace: ${NAMESPACE}
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: curl
      namespace: ${NAMESPACE}
      labels:
        app: curl
        service: curl
    spec:
      ports:
      - port: 80
        name: http
      selector:
        app: curl
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: curl
      namespace: ${NAMESPACE}
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: curl
      template:
        metadata:
          labels:
            app: curl
        spec:
          terminationGracePeriodSeconds: 0
          serviceAccountName: curl
          containers:
          - name: curl
            image: curlimages/curl
            command: ["/bin/sleep", "infinity"]
            imagePullPolicy: IfNotPresent
            volumeMounts:
            - mountPath: /etc/curl/tls
              name: secret-volume
          volumes:
          - name: secret-volume
            secret:
              secretName: curl-secret
              optional: true
    EOF
    ```

4. Export the name of the `curl` Pod:
   
    ```Shell/Bash
    export SOURCE_POD=$(kubectl get pod -n "$NAMESPACE" -l app=curl -o jsonpath={.items..metadata.name})
    ```

### Add a hostname to the mesh

1. Define a ServiceEntry which adds the kyma-project.io hostname to the mesh:

    ```Shell/Bash
    kubectl apply -f - <<EOF
    apiVersion: networking.istio.io/v1
    kind: ServiceEntry
    metadata:
      name: kyma-project
      namespace: $NAMESPACE
    spec:
      hosts:
      - kyma-project.io
      ports:
      - number: 443
        name: tls
        protocol: TLS
      resolution: DNS
    EOF
    ```

### Create an egress Gateway, DestinationRule, and VirtualService to direct traffic

1. Create an egress Gateway:

    ```Shell/Bash
    kubectl apply -f - <<EOF
    apiVersion: networking.istio.io/v1
    kind: Gateway
    metadata:
      name: istio-egressgateway
      namespace: ${NAMESPACE}
    spec:
      selector:
        istio: egressgateway
      servers:
      - port:
          number: 443
          name: tls
          protocol: TLS
        hosts:
        - kyma-project.io
        tls:
          mode: PASSTHROUGH
    EOF
    ```
2. Create a DestinationRule:

    ```Shell/Bash
    kubectl apply -f - <<EOF
    apiVersion: networking.istio.io/v1
    kind: DestinationRule
    metadata:
      name: egressgateway-for-kyma-project
      namespace: ${NAMESPACE}
    spec:
      host: istio-egressgateway.istio-system.svc.cluster.local
      subsets:
      - name: kyma-project
    EOF
    ```

3. Create a VirtualService:

    ```Shell/Bash
    kubectl apply -f - <<EOF
    apiVersion: networking.istio.io/v1
    kind: VirtualService
    metadata:
      name: direct-kyma-project-through-egress-gateway
      namespace: ${NAMESPACE}
    spec:
      hosts:
      - kyma-project.io
      gateways:
      - mesh
      - istio-egressgateway
      tls:
      - match:
        - gateways:
          - mesh
          port: 443
          sniHosts:
          - kyma-project.io
        route:
        - destination:
            host: istio-egressgateway.istio-system.svc.cluster.local
            subset: kyma-project
            port:
              number: 443
      - match:
        - gateways:
          - istio-egressgateway
          port: 443
          sniHosts:
          - kyma-project.io
        route:
        - destination:
            host: kyma-project.io
            port:
              number: 443
          weight: 100
    EOF
    ```

### Send an HTTPS request to the website

To test your configutation, send an HTTPS request to the `kyma-project.io` website and check the Istio egress Gateway's logs.

1. Send an HTTPS request to the Kyma project website:

    ```Shell/Bash
    kubectl exec -n "$NAMESPACE" "$SOURCE_POD" -c curl -- curl -sSL -o /dev/null -D - https://kyma-project.io
    ```

    If successful, you get a response from the website similar to this one:

    ```Shell/Bash
    HTTP/2 200
    accept-ranges: bytes
    age: 203
    ...
    ```

2. Check the logs of the Istio egress Gateway:

    ```Shell/Bash
    kubectl logs -l istio=egressgateway -n istio-system
    ```
    
    If successful, the logs contain the request made by the egress Gateway:

    ```Shell/Bash
    {"requested_server_name":"kyma-project.io","upstream_cluster":"outbound|443||kyma-project.io",[...]}
    ```

### Enhance security by implementing NetworkPolicies

By default, Istio cannot securely enforce that egress traffic is routed through the Istio egress Gateway. It only enables the flow through sidecar proxies.

However, you can use Kubernetes NetworkPolicies to restrict namespace traffic, so it only passes through the Istio egress Gateway. NetworkPolicies are the Kubernetes method for enforcing traffic rules within a namespace.

> Support for NetworkPolicies depends on the Kubernetes CNI plugin used in the cluster. By default, SAP BTP, Kyma runtime uses the CNI configuration provided and managed by Gardener, which supports NetworkPolicies. However, if you’ve made any changes, make sure to check the relevant documentation.

> In Gardener-based clusters, such as SAP BTP, Kyma runtime, the Network Policy restricting DNS traffic may not work as expected. It is due to the local DNS service used in discovery working outside the CNI. In such cases, define the IP CIDR of the `kube-dns` service in the NetworkPolicy's **ipBlock** section to allow proper DNS resolution.

1. Fetch the IP address of the `kube-dns` Service:
    ```Shell/Bash
    export KUBE_DNS_ADDRESS=$(kubectl get svc -n kube-system kube-dns -o jsonpath='{.spec.clusterIP}')
    ```
2. Create a NetworkPolicy with the fetched IP address in the **ipBlock** section. The NetworkPolicy allows only egress
   traffic to the Istio egress Gateway, blocking all other egress traffic.
    ```Shell/Bash
    kubectl apply -f - <<EOF
    apiVersion: networking.k8s.io/v1
    kind: NetworkPolicy
    metadata:
        name: network-policy-allow-egress-traffic
        namespace: ${NAMESPACE}
    spec:
        egress:
        - ports:
          - port: 53
            protocol: UDP
          to:
          - namespaceSelector:
              matchLabels:
                  kubernetes.io/metadata.name: kube-system
          - ipBlock:
              cidr: ${KUBE_DNS_ADDRESS}/32
        - to:
          - namespaceSelector:
              matchLabels:
                  kubernetes.io/metadata.name: istio-system
        podSelector: {}
        policyTypes:
        - Egress
    EOF
    ```
3. Send an HTTPS request to the Kyma project website:
   
    ```Shell/Bash
    kubectl exec -n "$NAMESPACE" "$SOURCE_POD" -c curl -- curl -sSL -o /dev/null -D - https://kyma-project.io
    ```
   
    If successful, you get a response from the website similar to this one:
   
    ```Shell/Bash
    HTTP/2 200
    accept-ranges: bytes
    age: 203
    ...
    ```
4. Send an HTTPS request to an external website:
    ```Shell/Bash
    kubectl exec -n "$NAMESPACE" "$SOURCE_POD" -c curl -- curl -sSL -o /dev/null -D - https://www.google.com
    ```
   
    The request should fail with an error message similar to this one:
    
    ```Shell/Bash
    curl: (35) Recv failure: Connection reset by peer
    command terminated with exit code 35
    ```
    You have successfully secured the egress traffic in your namespace using Istio egress Gateway and Kubernetes NetworkPolicies.
