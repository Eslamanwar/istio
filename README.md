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







# Using Linkerd
## prerequisite

- Check that your k8s version running 1.13 or later 
```
kubectl version --short
```
 


## Linkerd installation
```
curl -sL https://run.linkerd.io/install | sh
export PATH=$PATH:$HOME/.linkerd2/bin
linkerd version

# Add the linkerd CLI to your path with:
export PATH=$PATH:/home/admin-eslam/.linkerd2/bin


linkerd check --pre                     # validate that Linkerd can be installed
linkerd install | kubectl apply -f -    # install the control plane into the 'linkerd' namespace
linkerd check                           # validate everything worked!
linkerd dashboard                       # launch the dashboard

```
- injecting Linkerd into your ingress controller's pods
```
kubectl get -n {ingress controller namespace } deploy  {ingress controller Deployment name}-o yaml \
  | linkerd inject - \
  | kubectl apply -f -
```


- There is some configuration i added in the your ingress annotation in your app in the helm chart , I added both http and grpc , and Nginx will ingnore the one that you are not using
```
    nginx.ingress.kubernetes.io/configuration-snippet: |
      proxy_set_header l5d-dst-override $service_name.$namespace.svc.cluster.local:$service_port;
      grpc_set_header l5d-dst-override $service_name.$namespace.svc.cluster.local:$service_port;

```

- I also addedd Annotation to your deployment file to enable Linkerd
```
      annotations:
        linkerd.io/inject: enabled
```

- if you have another deployment that you want to intergrate linkerd with:
```
kubectl get -n {Namespace} deploy {Deployment_Name}-o yaml \
  | linkerd inject - \
  | kubectl apply -f -
```



- Deploying Application
```
cd app-linkerd
helm install ./
```



## Watch it run

- Linkerd Dashboard
```
linkerd dashboard                       # launch the dashboard
```

- check that linkerd is added to existing services
```
linkerd -n {namespace} check --proxy
```


- to get a real-time view of which paths are being called:
```
linkerd -n {namespace} top deploy
```


- To go even deeper, we can use tap shows the stream of requests across a single pod
```
linkerd -n {namespace} tap deploy/web
```



- About things that happened in the past? Linkerd includes Grafana to visualize the metrics collected by Prometheus, and ships with some pre-configured dashboards. You can get to these by clicking the Grafana icon in the overview page.


























