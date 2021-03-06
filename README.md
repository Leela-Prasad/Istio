Istio is a Service Mesh

A service Mesh is an extra layer of software that we can deploy along with our cluster (eg: K8S Cluster).

In K8S if we want to communicate with other container then we will use the service name (which exposes network endpoint for that group of pods), However there is a problem with this type of orchestration i.e., we dont have a high level view on how interactions are happening between pods. This generally we will solve using logging or Tracing.

In Istio if a container wants to communicate with other container then it should go via Service Mesh, In this service mesh we can have a mesh logic as per the requirements like metrics, traffic management, security etc.

In Istio this service mesh is implemented by deploying a proxy container on each pod. So Group of all Proxies that are deployed in the pods are called Data Plane.

We will also have some Istio specific containers running in the cluster like Istio-Telemetry, Istio-pilot, Istio-tracing, This group of pods is called as Control Plane.

Running Istio:
We need a system with 8 GB RAM for running Istio.
In this course we will run Istio on minikube

sudo -i
sudo yum update
sudo yum install docker

curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl
chmod +x kubectl 
sudo mv ./kubectl /usr/bin/kubectl

curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
chmod +x minikube 
sudo mv ./minikube /usr/bin/

yum install conntrack
minikube start --vm-driver=none

Apply below files in sequence
1-istio-init.yaml
2-istio-minikube.yaml
3-kiali-secret.yaml
4-application-full-stack.yaml

All Istio defined pods, services etc will start on namespace "istio-system".

This file "4-application-full-stack.yaml" contains application specific pods, we need to run side car proxy or data plane in all of the containers
To enable this data plane we need to have a label on the namespace where we are running our application pods
In this course we will run all our application pods in default namespace
By Default no label will be there on default namespace

kubectl describe ns default
kubectl label namespace default istio-injection=enabled
kubectl describe ns default

we can have this kubectl command as a yaml file by generating automatically.
kubectl describe ns default -o yaml

*** After applying this label for each of the application pod we should see 2 containers running.

Proxy/Side Car/Envoy:
In Istio every pod will have a side car/proxy/Envoy
This proxy will have lot of features like telemetry, traffic management, security etc

This Proxy is not developed by Istio directly
This Proxy is developed by Envoy 
Then why can't we use Envoy directly insted of Istio as it can provide cross cutting concerns we are expecting.
because 
Envoy Proxy is developed to provide a capability of being deployed into many different types of clusters. This Envoy is not created for a kubernetes alone. So obvisouly this involves a massive and complex terminology of its own, and we need to map/Integrate these compoenents with kubernetes components to create a service mesh.

Istio is doing exactly the same by mapping/Integrating Envoy Components with K8S components.

*** What Istio will do is convert regular K8S Yaml files(via CRD's(Custom Resource Definition)) into Envoy configuration files automatically under the hood.

Istio Control Plane Components:
1. Galley:
Galley Reads our K8S YAML Files and transforms into an internal structure that Istio Understands.
This means Istio can work with non kubernetes environments such as consul as there will be different galley for ingestion.

2. Pilot:
Pilot is responsible for converting Istio configuration files into a format that Envoy can understand.
It is also responsible for propagating configuration to proxies/envoy.
This helps us not to work with Envoy configuration files directly.

3. Citadel
Citadel manages certificates.
With this we can enable TLS/SSL across our entire cluster.

4. Mixer
Mixer will get the telemetry from the proxies and get some sensible data for Dashboards such as Grafana
Mixer also have policy checks for security, rate limiting etc


*** From Istio 1.5 onwards all of the components like Pilot, Citadel, Galley were replaced with a single component called IstioD (Istio Deamon)

Telemetry:
In Istio to enable Telemetry 
1. we need to have a proxy running in each pod that you want to monitor
   this is done via label.
2. The control plane needs to be running i.e., Mixer, Telemetry, Citadel, Galley etc

You DONOT need any specific Istio YAML Configuration i.e., No Virtual Services, No Gateways etc.

If we want Jaeger traces to be grouped then we need to pass the certain specific headers for every outbound call.
This is want happens
when a request comes from UI it checks for x-request-id, if it not there then guid will be generated
When we make a call to another pod then this x-request-id should be forwarded otherwise Jaeger UI cannot understand that these requests belong to same group.


Traffic Management:

Canary Release:
Deploy a new software component but this new component is live for a percentage of requests (i.e., 10% requests).

while true; do curl http://13.233.4.25:30080/api/vehicles/driver/City%20Truck; echo; sleep 0.2; done


Virtual Services enables us to configure custom routing rules to the service mesh.
*** Virtual Service do this custom routing by configuring all proxies that are present in the pods, and this happens via istio-pilot.
*** once proxies are configured then when invoking a pod (where routing rules are applied) it knows that it has to go via Istio-Load-Balancer where Destination Rules(i.e, weighted routing 90%, 10%) applied.

Virtual Service and K8S Service are completely different and they do different jobs.
You only need Virtual Service when custom routing is necessary.
where as K8S Service is needed for Service Discovery for other pods in the K8S Cluster. 

Istio Load Balancer Tweaks:
We might need same user to use canary
we can achieve this using stickiness in load balancer.
** But Unfortunately this stickiness option will not work with weighted rules.

Without weighted rules we can use stickiness option in load balancer.
Below are the methods
1. use source ip
2. http header

In useSourceIp based on the source ip it will serve a canary.
In httpHeader based on a header name it will serve a canary.

*** But when we pass httpHeader from web application then that header may not available to the microservice that we are triggering (eg: staff service), so we need to propagate these headers across microservices chain.

while true; do curl http://13.232.163.82:30080/api/vehicles/driver/City%20Truck; echo; sleep 0.2; done
while true; do curl --header "myval: 192" http://13.232.163.82:30080/api/vehicles/driver/City%20Truck; echo; sleep 0.2; done


Istio Gateway:
When we apply canary release with weighted routing on the webapp it will not work because client will directly call the webapp K8S Service and K8S Service will load balance between the pods available.

Since client request is not going through proxy or envoy weighted routing will not be applicable.
To solve this we need to use a Edge Proxy so that client request can go through Edge Proxy where weighted routing configuration will be there.
In Istio this Edge Proxy is a Gateway.



while true; do curl -s http://52.66.220.132:30080 | grep title; echo; sleep 0.2; done


Dark Release:
Dark Release allow us to deploy a risky/untested service into the cluster and with a special header we can test the untested service.
This will allow us to avoid of maintaining staging/dark pod regions.
Avoid Staging/Dark Pod Regions has following Benefits
1. Cost Reduction (as we dont have staging region)
2. Most of the time we might miss some configuration issue when moving the service which has new feature to live region, This Dark Release will allow us to test directly in production with the same configuration as production :) so no configuration will be missed.

*** For this Dark Release to happen we need pass that custom header across micro services chain otherwise it will NOT work.


Fault Injection:
To test the stability of our Micro service architecture we have to inject faults via below methods
1. Bringing down a service 
2. injecting a delay in the service response

This way we can see how our systems behave during unexcepted situations.
This is called as Chaos Engineering.


Circuit Breaker:
Let us assume 
App A depends on App B 
App B depends on App C 
App C depends on App D 

Suppose if App D is processing request very slowly may be due to resource crunch or server crash or some other reason then in an applicaiton where there will be lot of requests.

App D will have lot of requests waiting in the queue for processing, this will create a back pressure on App D.
Since App D is processing requests very slowly APP C also will also have lot of requests waiting in the queue for processing
Since App C is processing requests very slowly APP B also will also have lot of requests waiting in the queue for processing
Since App B is processing requests very slowly APP A also will also have lot of requests waiting in the queue for processing

In this situation all the Applicaitons which are in the chain to App D will have lot of back pressure to process the request which may lead to crash, so one application slowness results to APP A,APP B, APP C, APP D to crash. This is called Cascading Failures.

So if we dont have a solution then one micro service slowness can result in crashing all microservices that are in the chain to that micro service.

To solve this we use a circuit breaker, so this circuit breaker will observe if there any consecutive failures for X number of times then circuit breaker will open to that microservice and any requests to that micro service will be rejected immediately.
This rejection can help that micro service to recover due to a pod restart or some intermittent issue.
This Rejction prevents back pressure to multiple micro service.
After a certain amount of time this circuit breaker will close and service will be connected to other microservices, if again it observers any consecutive failures for X number of times the CB will open again and this process continues. 

Hystrix is one of the circuit breaker provided by Netflix.
but Hystrix is doesn't have any active developments and is stopped by Netflix, Below are the reasons.

Hystrix is only for Java Applications(Not Cross Platform).
if we want to implement Circuit Breaker with Hystrix then we need to configure/code it inside micro services, so we don't have tooling to say which micro services have circuit breaker and which microservices are not having.

Istio provides a cross platform Circuit breaker and tooling to check whether a specific micro service have Circuit Breaker or not.
In Istio Circuit Breaker will be implemented at proxy level, so Application doesn't know anything about Circuit Breaker.

*** Istio will apply Circuit Breaker at Pod level, suppose if a pod is not responding then it will be removed (because CB is open), and routes the traffic to other pods of that service.
If all the pods of that service has CB open then only that service will be removed completely from other micro services.



while true; do curl http://13.234.119.204:31380/api/vehicles/driver/City%20Truck; echo; done

Mutual TLS:
In K8 Cluster communication happens between lot of micro services.
These micro services are on different nodes and these nodes might be in different data centers, So when one service invokes another service it has to cross the network via cables and is possible for Man in Middle Attack.
So it is mandatory to have secure communication between micro services, but it is very difficult to have all micro services to have this TLS Setup.

Istio by default provides this TLS layer between micro services in the cluster through Proxy layer, whenever one service invokes other service in the proxy layer it will automatically upgrade the connection layer to TLS with the help of certificates.
The services in the communication will do a Mutual TLS means service1 verifies service2 certificate and service2 verifies service1 certificate.
This Mutual TLS happens because of citadel component.

In Istio we have 2 types of mTLS
1. PERMISSIVE (default) 
2. STRICT 

suppose we have a service outside of istio which invokes a service within the istio system, so outside service will not be passed through proxy the communicaiton layer will not be upgraded to TLS until it provide certificate.
If it doesn't have certificate still Istio allows this communication to happen and we call this as PERMISSIVE.

But we can enforce every service communicate happen via TLS through STRICT mTLS option.


curl http://15.206.28.20:32000/vehicles/


Istio Installation:
To Install Istio through YAML files we need to generate Istio Configuration file so that we can keep it in source control, for this we need to install istioctl similar to kubectl.

 curl -L https://istio.io/downloadIstio | sh -

 This will install latest release of istioctl, but if we want to install specific version

 curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.5.2 sh -
 
 export PATH="$PATH:/root/istio-1.5.2/bin"

 To install istio with default profile
 istioctl mainfest apply

If we want to install istio with different profile then 
istioctl manifest apply --set profile=demo

then we can check pods in istio-system namespace
kubectl get pod -n istio-system

To add Kiali, grafana for default profile
istioctl manifest apply --set profile=default --set addComponents.kiali.enabled=true --set addComponents.grafana.enabled=true

To list down all profiles
istioctl profile list

To get entire profile dump
istioctl profile dump demo > raw_settings.yaml

we can't run this raw_settings.yaml file with kubectl as below because Kubernetes doesn't know about custom resource definitions of Istio.
kubectl apply -f raw_settings.yaml

To apply this entire we need to convert to kubectl format or we can execute below command
istioctl manifest apply -f raw_settings.yaml --set values.global.jwtPolicy=first-party-jwt


But we can also generate kubernetes specific yaml files also
istioctl manifest generate -f raw_settings.yaml --set values.global.jwtPolicy=first-party-jwt > istio_configuration.yaml

kubectl apply -f istio_configuration.yaml




master 172.20.61.240
node1 172.20.54.56
node2 172.20.33.227
node3 172.20.81.202


[ec2-user@ip-172-31-20-7 ~]$ kubectl get pod --all-namespaces -o wide
NAMESPACE     NAME                                                                   READY   STATUS    RESTARTS   AGE     IP              NODE                                           NOMINATED NODE   READINESS GATES
kube-system   dns-controller-776cdf4ff4-vjlfx                                        1/1     Running   0          2m52s   172.20.61.240   ip-172-20-61-240.ap-south-1.compute.internal   <none>           <none>
kube-system   etcd-manager-events-ip-172-20-61-240.ap-south-1.compute.internal       1/1     Running   0          2m40s   172.20.61.240   ip-172-20-61-240.ap-south-1.compute.internal   <none>           <none>
kube-system   etcd-manager-main-ip-172-20-61-240.ap-south-1.compute.internal         1/1     Running   0          2m45s   172.20.61.240   ip-172-20-61-240.ap-south-1.compute.internal   <none>           <none>
kube-system   kops-controller-k2x2p                                                  1/1     Running   0          112s    172.20.61.240   ip-172-20-61-240.ap-south-1.compute.internal   <none>           <none>
kube-system   kube-apiserver-ip-172-20-61-240.ap-south-1.compute.internal            1/1     Running   3          2m25s   172.20.61.240   ip-172-20-61-240.ap-south-1.compute.internal   <none>           <none>
kube-system   kube-controller-manager-ip-172-20-61-240.ap-south-1.compute.internal   1/1     Running   0          118s    172.20.61.240   ip-172-20-61-240.ap-south-1.compute.internal   <none>           <none>
kube-system   kube-dns-autoscaler-594dcb44b5-tf88b                                   1/1     Running   0          2m55s   100.96.1.3      ip-172-20-81-202.ap-south-1.compute.internal   <none>           <none>
kube-system   kube-dns-b84c667f4-grwsh                                               3/3     Running   0          2m55s   100.96.1.4      ip-172-20-81-202.ap-south-1.compute.internal   <none>           <none>
kube-system   kube-dns-b84c667f4-kkqjq                                               3/3     Running   0          91s     100.96.3.2      ip-172-20-33-227.ap-south-1.compute.internal   <none>           <none>
kube-system   kube-proxy-ip-172-20-33-227.ap-south-1.compute.internal                1/1     Running   0          73s     172.20.33.227   ip-172-20-33-227.ap-south-1.compute.internal   <none>           <none>
kube-system   kube-proxy-ip-172-20-54-56.ap-south-1.compute.internal                 1/1     Running   0          18s     172.20.54.56    ip-172-20-54-56.ap-south-1.compute.internal    <none>           <none>
kube-system   kube-proxy-ip-172-20-61-240.ap-south-1.compute.internal                1/1     Running   0          93s     172.20.61.240   ip-172-20-61-240.ap-south-1.compute.internal   <none>           <none>
kube-system   kube-proxy-ip-172-20-81-202.ap-south-1.compute.internal                1/1     Running   0          86s     172.20.81.202   ip-172-20-81-202.ap-south-1.compute.internal   <none>           <none>
kube-system   kube-scheduler-ip-172-20-61-240.ap-south-1.compute.internal            1/1     Running   0          2m38s   172.20.61.240   ip-172-20-61-240.ap-south-1.compute.internal   <none>           <none>





[ec2-user@ip-172-31-20-7 ~]$ kubectl get pod -o wide
NAME                                   READY   STATUS    RESTARTS   AGE   IP           NODE                                           NOMINATED NODE   READINESS GATES
api-gateway-6c879cd66-4sq5g            2/2     Running   0          75s   100.96.3.5   ip-172-20-33-227.ap-south-1.compute.internal   <none>           <none>
position-simulator-6564858dcc-s49bv    2/2     Running   0          75s   100.96.2.4   ip-172-20-54-56.ap-south-1.compute.internal    <none>           <none>
position-tracker-7d688c4c6c-kzlql      2/2     Running   0          75s   100.96.2.5   ip-172-20-54-56.ap-south-1.compute.internal    <none>           <none>
staff-service-7f5f487f66-svm8g         2/2     Running   0          75s   100.96.3.7   ip-172-20-33-227.ap-south-1.compute.internal   <none>           <none>
staff-service-risky-6c7d4d4c8c-mz9ck   2/2     Running   0          74s   100.96.1.8   ip-172-20-81-202.ap-south-1.compute.internal   <none>           <none>
vehicle-telemetry-649bb8c5c-tsrcv      2/2     Running   0          75s   100.96.2.6   ip-172-20-54-56.ap-south-1.compute.internal    <none>           <none>
webapp-54f689d777-4pctj                2/2     Running   0          75s   100.96.3.6   ip-172-20-33-227.ap-south-1.compute.internal   <none>           <none>
webapp-experimental-77b59b997f-n9sfv   2/2     Running   0          75s   100.96.1.9   ip-172-20-81-202.ap-south-1.compute.internal   <none>           <none>



[ec2-user@ip-172-31-20-7 ~]$ kubectl get pod -n istio-system -o wide
NAME                                   READY   STATUS    RESTARTS   AGE    IP           NODE                                           NOMINATED NODE   READINESS GATES
grafana-5cc7f86765-k6hds               1/1     Running   0          3m2s   100.96.1.5   ip-172-20-81-202.ap-south-1.compute.internal   <none>           <none>
istio-egressgateway-57999c5b76-tzh5g   1/1     Running   0          3m1s   100.96.1.6   ip-172-20-81-202.ap-south-1.compute.internal   <none>           <none>
istio-ingressgateway-968cdfcb5-vmptk   1/1     Running   0          3m1s   100.96.3.4   ip-172-20-33-227.ap-south-1.compute.internal   <none>           <none>
istio-tracing-8584b4d7f9-vnk5g         1/1     Running   0          3m1s   100.96.2.2   ip-172-20-54-56.ap-south-1.compute.internal    <none>           <none>
istiod-86798869b8-6v7t5                1/1     Running   0          3m     100.96.2.3   ip-172-20-54-56.ap-south-1.compute.internal    <none>           <none>
kiali-76f556db6d-5wrrp                 1/1     Running   0          3m     100.96.3.3   ip-172-20-33-227.ap-south-1.compute.internal   <none>           <none>
prometheus-6fd77b7876-gmqxx            2/2     Running   0          3m     100.96.1.7   ip-172-20-81-202.ap-south-1.compute.internal   <none>           <none>
