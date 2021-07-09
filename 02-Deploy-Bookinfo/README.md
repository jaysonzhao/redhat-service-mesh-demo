## Deploy A Bookinfo Store

### Prerequisite 
```bash
# Setup Project name
BOOKINFO_PROJECT=bookinfo && echo $BOOKINFO_PROJECT

# Get Istio Latest Release Version
ISTIO_RELEASE=$(curl --silent https://api.github.com/repos/istio/istio/releases/latest |grep -Po '"tag_name": "\K.*?(?=")') && echo $ISTIO_RELEASE
```

### 部署 bookinfo 於指定的專案中
```bash
# Create Project
oc new-project $BOOKINFO_PROJECT

# Add project into service mesh member
oc get smmr default -n istio-system -o json --export | jq '.spec.members += ["'"$BOOKINFO_PROJECT"'"]' | oc apply -n istio-system -f -

# Apply bookinfo
oc apply -n $BOOKINFO_PROJECT -f https://raw.githubusercontent.com/istio/istio/$ISTIO_RELEASE/samples/bookinfo/platform/kube/bookinfo.yaml

# Patch sidecar annotation into template in deployment object
for deployment in $(oc get deployments -o jsonpath='{.items[*].metadata.name}' -n $BOOKINFO_PROJECT);do
    oc -n $BOOKINFO_PROJECT patch deployment $deployment -p '{"spec":{"template":{"metadata":{"annotations":{"sidecar.istio.io/inject": "true"}}}}}'
done

# Patch the securitycontext
for deployment in $(oc get deployments -o jsonpath='{.items[*].metadata.name}' -n $BOOKINFO_PROJECT);do
    oc -n $BOOKINFO_PROJECT patch deployment $deployment --type=json -p '[{
  "op": "replace",
  "path": "/spec/template/spec/containers/0/securityContext/runAsUser",
  "value": 1000650008
 }]'
done

# Apply bookinfo gateway
oc apply -n $BOOKINFO_PROJECT -f https://raw.githubusercontent.com/istio/istio/$ISTIO_RELEASE/samples/bookinfo/networking/bookinfo-gateway.yaml
```

![](../images/02-deploy-bookinfo.gif)


### 獲得 Ingress Gateway 的網址，並測試連通性
```bash
# Get Ingress GW URL
GATEWAY_URL=$(oc -n istio-system get route istio-ingressgateway -o jsonpath='{.spec.host}') && echo $GATEWAY_URL

# Confirm the app is accessible from outside the cluster
while true; do sleep 1; curl -s http://${GATEWAY_URL}/productpage | grep -o "<title>.*</title>";done
```

![](../images/02-check-network-reachable.gif)


## Clean Up
```bash
oc delete -n $BOOKINFO_PROJECT -f https://raw.githubusercontent.com/istio/istio/$ISTIO_RELEASE/samples/bookinfo/networking/bookinfo-gateway.yaml
oc delete -n $BOOKINFO_PROJECT -f https://raw.githubusercontent.com/istio/istio/$ISTIO_RELEASE/samples/bookinfo/platform/kube/bookinfo.yaml
oc delete project $BOOKINFO_PROJECT
```
