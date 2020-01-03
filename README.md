# istio

## Installing Helm
```
cd /tmp
curl https://raw.githubusercontent.com/kubernetes/helm/master/scripts/get > install-helm.sh
chmod u+x install-helm.sh
./install-helm.sh

kubectl -n kube-system create serviceaccount tiller
kubectl create clusterrolebinding tiller --clusterrole cluster-admin --serviceaccount=kube-system:tiller
helm init --service-account tiller
```








## Deploying istio

- add istio to your helm repo:
```
cd ../istio_installation/
helm repo add istio.io https://storage.googleapis.com/istio-release/releases/1.4.2/charts/
helm repo update
```

- download the istio release:
```
curl -L https://istio.io/downloadIstio | sh -
cd istio-1.4.2
```

- Install the istio-init chart to bootstrap all the Istio’s CRDs:
```
helm install install/kubernetes/helm/istio-init --name istio-init --namespace istio-system
```

- wait until the pods is completed:
```
kubectl -n istio-system wait --for=condition=complete job --all
# or
kubectl get po -n istio-system --watch
```

- install the istio on k8s
```
helm install install/kubernetes/helm/istio --name istio --set global.configValidation=false  --namespace istio-system 
```




Verifying the installation:
```
kubectl get svc istio-ingressgateway -n istio-system
```


- Ensure the corresponding Kubernetes pods are deployed and have a STATUS of Running:
```
kubectl get pods -n istio-system
```


















## Deploying Application
```
cd ./app

## Generate a self-signed certificate for testing 

export CERT_NAME=example.com-tls
export KEY_FILE=example.com.csr
export CERT_FILE=example.com.crt
export HOST=example.com

openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout ${KEY_FILE} -out ${CERT_FILE} -subj "/CN=${HOST}/O=${HOST}"
kubectl create secret tls ${CERT_NAME} --key ${KEY_FILE} --cert ${CERT_FILE}


## i added variable in values file called (tls_secret_name) which should contain the name of the created k8s secret which contains the TLS certificate
helm install ./
```






## test the application with the istio Gateway
```
export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].port}')
export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].port}')

# as i changed the values file with hostname=example.com
curl -I -HHost:example.com http://$INGRESS_HOST:$INGRESS_PORT/
```




## Enable distributed-tracing with jaeger


- install the jaeger tracing dashboard
```
cd ../istio_installation/istio-1.4.2
./bin/istioctl manifest apply --set values.tracing.enabled=true --set values.tracing.ingress.enabled=true 
```
- it may give errors depending on your k8s cluster version but you can ignore it




- Accessing the dashboard
```
./bin/istioctl dashboard jaeger
```


- Generating traces:
To see trace data, you must send requests to your service. The number of requests depends on Istio’s sampling rate. You set this rate when you install Istio. The default sampling rate is 1%. You need to send at least 100 requests before the first trace is visible. 
```
export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT
for i in `seq 1 100`; do curl -s -o /dev/null http://$GATEWAY_URL/; done
```


## testing
![alt text](https://github.com/Eslamanwar/istio/blob/master/images/jaeger.png?raw=true)






## adding Default route for Authentication service
- All internal services within the mesh should naturally talk to each other.
- but istio simplifies configuration of service-level properties like circuit breakers, timeouts, and retries
- Set default routes for authentication-service
- define a proper route rule for the service in the VirtualService definition. For example, add a simple route rule:

```
cat << EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: authentication-service
spec:
  hosts:
  - authentication-service
  http:
  - route:
    - destination:
        host: authentication-service
        subset: v1
EOF
```












