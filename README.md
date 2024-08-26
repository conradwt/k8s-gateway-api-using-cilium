# K8s Gateway API Using Cilium

The purpose of this example is to provide instructions for running the K8s Gatewey API using Cilium.

## Software Requirements

- Cilium CLI v0.16.15 or newer

- Helm v3.15.2 or newer

- Kubernetes 1.31.0 or newer

- Minikube v1.33.1 or newer

- OrbStack v1.6.2 or newer

Note: This tutorial was updated on macOS 14.6.1. The below steps doesn't work with Docker Desktop v4.31.1
because it doesn't expose Linux VM IP addresses to the host OS (i.e. macOS).

## Tutorial Installation

1.  clone github repository

    ```zsh
    git clone https://github.com/conradwt/k8s-gateway-api-using-cilium.git
    ```

2.  change directory

    ```zsh
    cd k8s-gateway-api-using-cilium
    ```

3.  create Minikube cluster

    ```zsh
    minikube start --network-plugin=cni --cni=false --profile=gateway-api-cilium --nodes=3 --kubernetes-version=v1.31.0
    ```

4.  install Kubernetes Gateway API CRDs

    ```zsh
    kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.1.0/experimental-install.yaml
    ```

5.  enable Cilium Gateway API Controller

    ```zsh
    cilium install --version 1.16.1 \
     --set kubeProxyReplacement=true \
     --set gatewayAPI.enabled=true
    ```

6.  enable the Hubble UI

    ```zsh
    cilium hubble enable --ui
    ```

7.  install MetalLB

    ```zsh
    kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.8/config/manifests/metallb-native.yaml
    ```

8.  locate the K8s cluster's subnet

    ```zsh
    docker network inspect gateway-api-cilium | jq '.[0].IPAM.Config[0]["Subnet"]'
    ```

    The results should look something like the following:

    ```json
    "194.1.2.0/24",
    ```

    Then one can use an IP address range like the following:

    ```
    194.1.2.100-194.1.2.110
    ```

9.  create the `01-metallb-address-pool.yaml` file

    ```zsh
    cp 01-metallb-address-pool.yaml.example 01-metallb-address-pool.yaml
    ```

10. update the `01-metallb-address-pool.yaml`

    ```yaml
    apiVersion: metallb.io/v1beta1
    kind: IPAddressPool
    metadata:
      name: demo-pool
      namespace: metallb-system
    spec:
      addresses:
        - 194.1.2.100-194.1.2.110
    ```

    Note: The IP range needs to be in the same range as the K8s cluster, `gateway-api-cilium`.

11. apply the address pool manifest

    ```zsh
    kubectl apply -f 01-metallb-address-pool.yaml
    ```

12. apply Layer 2 advertisement manifest

    ```zsh
    kubectl apply -f 02-metallb-advertise.yaml
    ```

13. deploy the demo app

    ```zsh
    kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.11/samples/bookinfo/platform/kube/bookinfo.yaml
    ```

14. deploy the Cilium Gateway

    ```zsh
    kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/1.16.1/examples/kubernetes/gateway/basic-http.yaml
    ```

15. output info about the gateway resource

    ```zsh
    kubectl get gateway my-gateway
    ```

    The results should look something like the following:

    ```zsh
    NAME         CLASS    ADDRESS          PROGRAMMED   AGE
    my-gateway   cilium   192.168.49.100   True         8s
    ```

16. output info about the service resource

    ```zsh
    kubectl get svc cilium-gateway-my-gateway
    ```

    The results should look something like the following:

    ```zsh
    NAME                        TYPE           CLUSTER-IP       EXTERNAL-IP      PORT(S)        AGE
    cilium-gateway-my-gateway   LoadBalancer   10.102.213.185   192.168.49.100   80:32085/TCP   18s
    ```

17. populate $GATEWAY_IP for future commands:

    ```zsh
    export GATEWAY_IP=$(kubectl get gateway my-gateway -o jsonpath='{.status.addresses[0].value}')
    echo $GATEWAY_IP
    ```

18. test the routing rule

    ```zsh
    curl --fail -s http://"${GATEWAY_IP}"/details/1 | jq
    ```

    The results should look something like the following:

    ```json
    {
      "id": 1,
      "author": "William Shakespeare",
      "year": 1595,
      "type": "paperback",
      "pages": 200,
      "publisher": "PublisherA",
      "language": "English",
      "ISBN-10": "1234567890",
      "ISBN-13": "123-1234567890"
    }
    ```

    ```zsh
    curl -v -H 'magic: foo' http://"${GATEWAY_IP}"\?great\=example
    ```

    The results should look something like the following:

    ```text
    *   Trying 192.168.49.100:80...
    * Connected to 192.168.49.100 (192.168.49.100) port 80
    > GET /?great=example HTTP/1.1
    > Host: 192.168.49.100
    > User-Agent: curl/8.9.1
    > Accept: */*
    > magic: foo
    >
    * Request completely sent off
    < HTTP/1.1 200 OK
    < content-type: text/html; charset=utf-8
    < content-length: 1683
    < server: envoy
    < date: Mon, 26 Aug 2024 04:32:00 GMT
    < x-envoy-upstream-service-time: 34
    <
    <!DOCTYPE html>
    <html>
      <head>
        <title>Simple Bookstore App</title>
    <meta charset="utf-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1">

    <!-- Latest compiled and minified CSS -->
    <link rel="stylesheet" href="static/bootstrap/css/bootstrap.min.css">

    <!-- Optional theme -->
    <link rel="stylesheet" href="static/bootstrap/css/bootstrap-theme.min.css">

      </head>
      <body>


    <p>
        <h3>Hello! This is a simple bookstore application consisting of three services as shown below</h3>
    </p>

    <table class="table table-condensed table-bordered table-hover"><tr><th>name</th><td>http://details:9080</td></tr><tr><th>endpoint</th><td>details</td></tr><tr><th>children</th><td><table class="table table-condensed table-bordered table-hover"><tr><th>name</th><th>endpoint</th><th>children</th></tr><tr><td>http://details:9080</td><td>details</td><td></td></tr><tr><td>http://reviews:9080</td><td>reviews</td><td><table class="table table-condensed table-bordered table-hover"><tr><th>name</th><th>endpoint</th><th>children</th></tr><tr><td>http://ratings:9080</td><td>ratings</td><td></td></tr></table></td></tr></table></td></tr></table>

    <p>
        <h4>Click on one of the links below to auto generate a request to the backend as a real user or a tester
        </h4>
    </p>
    <p><a href="/productpage?u=normal">Normal user</a></p>
    <p><a href="/productpage?u=test">Test user</a></p>



    <!-- Latest compiled and minified JavaScript -->
    <script src="static/jquery.min.js"></script>

    <!-- Latest compiled and minified JavaScript -->
    <script src="static/bootstrap/js/bootstrap.min.js"></script>

      </body>
    </html>
    * Connection #0 to host 192.168.49.100 left intact
    ```

19. tear down the cluster

    ```zsh
    minikube delete --profile gateway-api-cilium
    ```

## References

- https://docs.cilium.io/en/latest/network/servicemesh/gateway-api/gateway-api/#what-is-gateway-api

## Troubleshooting

- https://docs.cilium.io/en/latest/network/l2-announcements/#troubleshooting
