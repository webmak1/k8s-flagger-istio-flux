# Kubernetes + Flagger + Flux + Istio

**Based on materials:**

* Web Pages: [https://ruzickap.github.io/k8s-flagger-istio-flux](https://ruzickap.github.io/k8s-flagger-istio-flux)
* YouTube: [https://youtu.be/ot4SvFZWJuE](https://youtu.be/ot4SvFZWJuE)


<br/>

## Prepared env


```
$ {
minikube --profile my-profile config set memory 8192
minikube --profile my-profile config set cpus 4

minikube --profile my-profile config set vm-driver virtualbox
// minikube --profile my-profile config set vm-driver docker

minikube --profile my-profile config set kubernetes-version v1.16.1
minikube start --profile my-profile
}
```

<br/>

    // Remove minikube
    // $ minikube --profile my-profile stop && minikube --profile my-profile delete

<br/>

    $ kubectl version --short
    Client Version: v1.18.1
    Server Version: v1.16.1


<br/>

### Setup istioctl on localhost

    $ curl -L https://istio.io/downloadIstio | sh - && chmod +x $HOME/istio-1.5.1/bin/istioctl && sudo mv $HOME/istio-1.5.1/bin/istioctl /usr/local/bin/

<br/>

### Run istio services in minikube

https://istio.io/docs/setup/additional-setup/config-profiles/


    $ istioctl manifest apply --set profile=default

<br/>

    $ kubectl label namespace default istio-injection=enabled



<br/>

### Add Metal LB

Metal LB help recieve external IP

<br/>

    $ kubectl apply -f https://raw.githubusercontent.com/google/metallb/v0.8.3/manifests/metallb.yaml

<br/>

    $ minikube --profile my-profile ip
    192.168.99.105

<br/>

```
$ cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: custom-ip-space
      protocol: layer2
      addresses:
      - 192.168.99.105/28
EOF
```

<br/>

### [Part 02 - Install Flux]

same as here 

https://github.com/webmakaka/flux-get-started

except repo should be

github.com/webmakaka/k8s-flux-repository


<br/>

### [Part 04 - Install Flagger]

    $ helm repo add flagger https://flagger.app

    $ kubectl apply -f https://raw.githubusercontent.com/weaveworks/flagger/master/artifacts/flagger/crd.yaml

    // $ helm search repo -l flagger/flagger
    $ helm search repo flagger/flagger


    // --version 0.23.0 \

    $ helm install flagger  flagger/flagger  --wait \
    --version 0.18.4 \
    --namespace=istio-system \
    --set crd.create="false" \
    --set logLevel="debug" \
    --set meshProvider="istio" \
    --set metricsServer="http://prometheus:9090"

    $ helm search repo flagger/grafana

    $ helm install flagger-grafana flagger/grafana --wait \
    --version 1.4.0 \
    --namespace=istio-system \
    --set password=admin \
    --set url=http://prometheus:9090 \
    --set user=admin


<br/>

    $ export MY_DOMAIN=webmakaka.com

```
$ cat << EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: flagger-grafana-gateway
  namespace: istio-system
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http-flagger-grafana
      protocol: HTTP
    hosts:
    - flagger-grafana.${MY_DOMAIN}
#  - port:
#      number: 443
#      name: https-flagger-grafana
#      protocol: HTTPS
#    hosts:
#    - flagger-grafana.${MY_DOMAIN}
#    tls:
#      credentialName: ingress-cert-${LETSENCRYPT_ENVIRONMENT}
#      mode: SIMPLE
#      privateKey: sds
#      serverCertificate: sds
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: flagger-grafana-virtual-service
  namespace: istio-system
spec:
  hosts:
  - flagger-grafana.${MY_DOMAIN}
  gateways:
  - flagger-grafana-gateway
  http:
  - route:
    - destination:
        host: flagger-grafana.istio-system.svc.cluster.local
        port:
          number: 80
EOF
```

<br/>

    // TO check

    $ kubectl -n istio-system describe gateway.networking.istio.io/flagger-grafana-gateway 

    $ kubectl -n istio-system get gateway
    NAME                      AGE
    flagger-grafana-gateway   63m
    ingressgateway            71m

    $ kubectl -n istio-system describe virtualservice.networking.istio.io/flagger-grafana-virtual-service

    $ kubectl -n istio-system get virtualservice
    NAME                              GATEWAYS                    HOSTS                             AGE
    flagger-grafana-virtual-service   [flagger-grafana-gateway]   [flagger-grafana.webmakaka.com]   62m


<br/>

### [Part 05 - Canary deployment using Flagger]

Update your repo  

Here is mine:  
https://github.com/webmakaka/k8s-flux-repository

replace webmakaka.com on your domain


<br/>

    $ fluxctl --k8s-fwd-ns=fluxcd sync


<br/>

    $ kubectl get service -n istio-system istio-ingressgateway


<br/>

    $ sudo vi /etc/hosts

<br/>

```
#---------------------------------------------------------------------
# Minikube
#---------------------------------------------------------------------

192.168.99.96 webmakaka.com
192.168.99.96 podinfo.webmakaka.com
192.168.99.96 flagger-grafana.webmakaka.com
```

<br/>

 http://podinfo.webmakaka.com


<br/>

    $ kubectl get pods
    NAME                       READY   STATUS    RESTARTS   AGE
    podinfo-7c77d7fd9d-dkmn7   2/2     Running   0          3m40s


<br/>

<!--

### 

    $ minikube --profile my-profile ip
    192.168.99.109

    $ kubectl -n istio-system get services 

    $ curl 192.168.99.109:31193

-->

<br/>

    $ kubectl -n istio-system describe gateway ingressgateway

<br/>

### Checks

    $ kubectl describe canaries.flagger.app podinfo

    $ kubectl get deployment

    $ kubectl get services

    $ kubectl describe virtualservices.networking.istio.io podinfo

    $ kubectl describe destinationrules.networking.istio.io


<br/>

### Add new image to registry


```
$ fluxctl --k8s-fwd-ns fluxcd  list-images --workload default:deployment/podinfo
WORKLOAD                    CONTAINER  IMAGE              CREATED
default:deployment/podinfo  podinfo    webmakaka/podinfo  
                                       '-> 0.4.12         19 Apr 20 06:48 UTC
                                           latest         18 Apr 20 04:25 UTC
                                           0.4.11         16 Apr 20 22:23 UTC
                                           0.4.10         16 Apr 20 22:21 UTC
                                           stg-5bkv94wh   16 Apr 20 22:18 UTC
                                           dev-mmbudouy   16 Apr 20 22:11 UTC
```

<br/>

    $ while true; do curl -s http://podinfo.webmakaka.com | jq .message; sleep 3; done

    $ while true; do kubectl get canary/podinfo -o json | jq .status; sleep 2; done

    $ kubectl -n istio-system logs deployment/flagger -f | jq .msg

    $ while true ; do kubectl get canaries; sleep 3; done

    $ watch curl -s http://podinfo.webmakaka.com/status/500

---

<strong>Marley</strong>

<a href="https://webmakaka.com"><strong>WebMakaka</strong></a>
