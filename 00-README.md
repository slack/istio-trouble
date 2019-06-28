# Istio Trouble

## Installation

```console
helm repo add istio.io https://storage.googleapis.com/istio-release/releases/1.2.0/charts/
helm repo update
helm fetch istio.io/istio-init

kubectl create namespace istio-system
#helm template istio-init-1.2.0.tgz --name istio-init --namespace istio-system | kubectl apply -f -


helm fetch istio.io/istio
tar --strip-components=1 -xvzf istio-1.2.0.tgz istio/values-istio-demo.yaml

helm template istio-1.2.0.tgz --name istio --namespace istio-system \
    --set global.proxy.accessLogFile="/dev/stdout" \
    --set kiali.enabled=true \
    --set "kiali.dashboard.jaegerURL=http://jaeger-query:16686" \
    --set "kiali.dashboard.grafanaURL=http://grafana:3000" \
    --values values-istio-demo.yaml | kubectl apply -f -
```

## Applications

### Basic app

```console
kubectl create ns trouble
kns trouble
kubectl apply -f manifests/01-podinfo-blue.yaml
kubectl apply -f manifests/02-podinfo-blue-service.yaml

kubectl get svc,ep,po

open http://blue.trouble.slack.io/  # should see timeout

cat manifests/03-gateway.yaml
kubectl apply -f manifests/03-gateway.yaml

cat manifests/04-virtualservice-blue.yaml
kubectl apply -f manifests/04-virtualservice-blue.yaml

open http://blue.trouble.slack.io/ 
```

### Second app

```console
kubectl apply -f manifests/05-podinfo-green.yaml
open http://green.trouble.slack.io/ # will be broken

kubectl get svc,gateway,virtualservice

kubectl apply -f manifests/06-patch-gateway.yaml
kubectl diff -f manifests/03-gateway.yaml

open http://green.trouble.slack.io/
```

### Third app: blue & green

```console
# app: blue & green
kubectl apply -f manifests/07-podinfo-both.yaml
open http://app.trouble.slack.io 

kubectl get gateway,virtualservice
```

### Traffic shifting

```console
kubectl apply -f manifests/08-podinfo-weighted.yaml

hey -z 600m -q 50 -c 15 http://app.trouble.slack.io

hey -z 600m -q 20 -c 15 http://blue.trouble.slack.io
hey -z 600m -q 20 -c 15 http://green.trouble.slack.io

kubectl -n istio-system port-forward svc/grafana 3000
kubectl -n istio-system port-forward svc/kiali 20001
```

### External services

```console
# confirm ALLOW_ANY
kubectl get configmap istio -n istio-system -o yaml | grep -o "mode: ALLOW_ANY"
# kubectl get configmap istio -n istio-system -o yaml | sed 's/mode: REGISTRY_ONLY/mode: ALLOW_ANY/g' | kubectl replace -n istio-system -f -

kubectl apply -f manifests/09-curler.yaml
kubectl logs deploy/curler -c curler --since 2m -f # works
kubectl delete -f manifests/09-curler.yaml

# switch to REGISTRY_ONLY mode
kubectl get configmap istio -n istio-system -o yaml | sed 's/mode: ALLOW_ANY/mode: REGISTRY_ONLY/g' | kubectl replace -n istio-system -f -


kubectl apply -f manifests/09-curler.yaml
kubectl logs deploy/curler -c curler --since 2m -f # busted

kubectl apply -f manifests/10-external-service.yaml

kubectl logs deploy/curler -c curler --since 2m -f # works!
```

```
kubectl get pod -l istio=egressgateway -n istio-system
```
