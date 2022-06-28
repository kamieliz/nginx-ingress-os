# NGINX Ingress Controller on OpenShift
This lab covers installing NGINX Ingress Controller on the OpenShift Container Platform and exposing a TCP service for load balancing.

# Table of Contents
1. Accessing OpenShift
2. Manual Installation
3. Installation using the OLM
4. Deploying NGINX Ingress Operator
5. Deploying TCP service

# NGINX Ingress Controller on OpenShift

## Getting Started

Login with XRDP in the ocp-provisioner node.

[picture of VNC screen]

## Accessing Openshift

### Using the CLI

Login via the terminal

`[cloud-user@ocp-provisioner ~]$ oc login -u f5admin -p f5admin`

### Using the UI

Open a browser with https://console-openshift-console.apps.ocp.f5-udf.com/

Use the HTTP auth provider

![Image.png](https://res.craft.do/user/full/d367a179-adcb-7ce8-0b02-ba52d2a7c917/doc/F7779AE5-6B4A-46F8-A1A7-EE43F5EFE81D/94D70987-375B-4086-8155-6C6767057BED_2/Image.png)

Login with the credentials f5admin/f5admin

![Image.png](https://res.craft.do/user/full/d367a179-adcb-7ce8-0b02-ba52d2a7c917/doc/F7779AE5-6B4A-46F8-A1A7-EE43F5EFE81D/2BA3166A-B301-4F86-80C0-8062F71E2512_2/Image.png)

## Manual Installation

This will deploy the operator in the `nginx-ingress-operator-system` namespace.

Create a new namespace for running your TCP service

`oc create ns nginx-ingress-operator-system`

Set terminal configuration context to your new namespace

`oc config set-context —current —namespace=nginx-ingress-operator-system`

1. Deploy the Operator and associated resources:
   1. Clone the `nginx-ingress-operator` repo and checkout the latest stable tag:

```other
git clone https://github.com/nginxinc/nginx-ingress-helm-operator/
cd nginx-ingress-helm-operator/
git checkout v1.0.0
```

   2. `Openshift` To deploy the Operator and associated resources to an OpenShift environment, run:

```other
make deploy IMG=nginx/nginx-ingress-operator:1.0.0
```

2. Check that the Operator is running:

```other
oc get deployments -n nginx-ingress-operator-system

NAME                                        READY   UP-TO-DATE   AVAILABLE   AGE
nginx-ingress-operator-controller-manager   1/1     1            1           15s
```

3. `Openshift` Additional steps:

In order to deploy NGINX Ingress Controller instances into OpenShift environments, a new SCC is required to be created on the cluster which will be used to bind the specific required capabilities to the NGINX Ingress service account(s). To do so, please run the following command (assuming you are logged in with administrator access to the cluster):

`oc apply -f` [`https://raw.githubusercontent.com/nginxinc/nginx-ingress-helm-operator/main/resources/scc.yaml`](https://raw.githubusercontent.com/nginxinc/nginx-ingress-helm-operator/main/resources/scc.yaml)

## Installing using the OLM

This installation method is the recommended way for Openshift users. **Note**: Openshift version must be 4.2 or higher.

**Note: The `nginx-ingress-operator` supports `Basic Install` only - we do not support auto-updates. When you are installing the Operator using the OLM, the auto-update feature should be disabled to avoid breaking changes being auto-applied. In OpenShift, this can be done by setting the `Approval Strategy` to `Manual`. Please see the** [**Operator SDK docs**](https://sdk.operatorframework.io/docs/advanced-topics/operator-capabilities/operator-capabilities/)** for more details on the Operator Capability Levels.**

The NGINX Ingress Operator is a [RedHat certified Operator](https://connect.redhat.com/en/partner-with-us/red-hat-openshift-operator-certification).

![openshift1.png](https://github.com/nginxinc/nginx-ingress-helm-operator/raw/v1.0.0/docs/images/openshift1.png)

1. In the Openshift dashboard, click `Operators` > `Operator Hub` in the left menu and use the search box to type `nginx ingress`:

![openshift2.png](https://github.com/nginxinc/nginx-ingress-helm-operator/raw/v1.0.0/docs/images/openshift2.png)

2. Click the `NGINX Ingress Operator` and click `Install`:

![openshift3.png](https://github.com/nginxinc/nginx-ingress-helm-operator/raw/v1.0.0/docs/images/openshift3.png)

3. Click `Subscribe`:

Openshift will install the NGINX Ingress Operator:

![openshift4.png](https://github.com/nginxinc/nginx-ingress-helm-operator/raw/v1.0.0/docs/images/openshift4.png)

### Additional steps:

In order to deploy NGINX Ingress Controller instances into OpenShift environments, a new SCC is required to be created on the cluster which will be used to bind the specific required capabilities to the NGINX Ingress service account(s). To do so, please run the following command (assuming you are logged in with administrator access to the cluster):

`oc apply -f` [`https://raw.githubusercontent.com/nginxinc/nginx-ingress-helm-operator/main/resources/scc.yaml`](https://raw.githubusercontent.com/nginxinc/nginx-ingress-helm-operator/main/resources/scc.yaml)

## Deploying NGINX Ingress Operator

Now to use NGINX Ingress Controller to expose a TCP service, you have to allow global configurations in the NGINX Ingress Deployment

change directory back to home

`cd ~`

open file that contains YAML configuration for NGINX Plus Ingress Controller

`nano 1_kubernetes-ingress/deployments/deployment/1_nginx-plus-ingress_custom.yaml`

uncomment the section that says:

`-global-configuration=$(POD_NAMESPACE)/nginx-configuration`

If section was changed run the command:

`oc apply -f 1_kubernetes-ingress/deployments/deployment/1_nginx_plus-ingress_custom.yaml`

To create the nginx-ingress service that allows access to tcp port 81

`nano nodeport_custom.yaml`

```other
apiVersion: v1
kind: Service
metadata:
  name: nginx-ingress
  namespace: nginx-ingress
  labels:
    app: nginx-ingress
spec:
  externalTrafficPolicy: Local
  type: NodePort
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
    protocol: TCP
    name: http
  - port: 81
    targetPort: 81
    nodePort: 30081
    protocol: TCP
    name: transport-tcp
  - port: 443
    targetPort: 443
    nodePort: 30443
    protocol: TCP
    name: https
  - port: 8081
    targetPort: 8081
    nodePort: 32081
    protocol: TCP
    name: readiness-port
  - port: 8080
    targetPort: 8080
    nodePort: 32080
    protocol: TCP
    name: dashboard
  - port: 9113
    targetPort: 9113
    nodePort: 32113
    protocol: TCP
    name: prometheus
  selector:
    app: nginx-ingress
```

`oc apply -f nodeport_custom.yaml`

## Deploying TCP service

Create a new namespace for running your TCP service

`oc create ns tcp-test`

Set terminal configuration context to your new namespace

`oc config set-context —current —namespace=tcp-test`

Apply the configuration for our example service

`Oc apply -f` [`https://raw.githubusercontent.com/nginxinc/kubernetes-ingress/main/examples/custom-resources/basic-configuration/cafe.yaml`](https://raw.githubusercontent.com/nginxinc/kubernetes-ingress/main/examples/custom-resources/basic-configuration/cafe.yaml)

```other
apiVersion: apps/v1
kind: Deployment
metadata:
  name: coffee
spec:
  replicas: 2
  selector:
    matchLabels:
      app: coffee
  template:
    metadata:
      labels:
        app: coffee
    spec:
      containers:
      - name: coffee
        image: nginxdemos/nginx-hello:plain-text
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: coffee-svc
spec:
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
    name: http
  selector:
    app: coffee
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tea
spec:
  replicas: 1
  selector:
    matchLabels:
      app: tea 
  template:
    metadata:
      labels:
        app: tea 
    spec:
      containers:
      - name: tea 
        image: nginxdemos/nginx-hello:plain-text
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: tea-svc
spec:
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
    name: http
  selector:
    app: tea
```

Once that is applied you can check your pods and see coffee and tea running

`oc get pods`

Now to expose the TCP service we must apply a configuration for a GlobalConfiguration resource.

`nano globalconfigurationresource.yaml`

```other
apiVersion: k8s.nginx.org/v1alpha1
kind: GlobalConfiguration
metadata:
  name: nginx-configuration
  namespace: nginx-ingress
spec:
  listeners:
  - name: coffee-tcp
    port: 81
    protocol: TCP
```

Apply this configuration us oc apply

`oc apply -f globalconfigurationresource.yaml`

To expose a TCP service, you need to create a TransportServer resource. This allows you to configure TCP, UDP, and TLS Passthrough load balancing. For TCP this resource must be used in conjunction with the GlobalConfiguration resource.

`nano transport-server.yaml`

```other
apiVersion: k8s.nginx.org/v1alpha1
kind: TransportServer
metadata:
  name: coffee-tcp
spec:
  listener:
    name: coffee-tcp
    protocol: TCP
  upstreams:
  - name: coffee-service
    service: coffee-svc
    port: 80
  action:
    pass: coffee-service
```

`oc apply -f ts.yaml`

Once the transport server configuration is applied, grab the nginx-ingress pod name to run the required configuration to expose a tcp service

`oc exec -n nginx-ingress nginx-ingress-[unique name] — nginx -T | grep ‘coffee-tcp’ -C 6`

To access the listener from external

`oc get svc nginx-ingress -n nginx-ingress`

you can go to NGINX Dashboard to see the TCP/UDP upstreams

![image](https://user-images.githubusercontent.com/4666871/176065586-c4062780-5a67-4521-b326-3bc4440792a3.png)
![image](https://user-images.githubusercontent.com/4666871/176065554-64f4b760-619f-4953-9bc6-4871f7b17ea3.png)
You can use the bookmark [http://10.1.1.6:30081/](http://10.1.1.6:30081/) to access the service via browser. Each time you open a new incognito browser you will get loadbalanced to a different POD (you see the POD hash changing under Server name:)

![image](https://user-images.githubusercontent.com/4666871/176065508-63bbeac1-778f-438c-84d7-12111978c46a.png)
