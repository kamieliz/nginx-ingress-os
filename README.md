# NGINX Ingress Controller on OpenShift
This lab covers installing NGINX Ingress Controller on the OpenShift Container Platform and exposing a TCP service for load balancing.

# Table of Contents
1. [Getting Started](#getting-started)
2. [Accessing OpenShift](#accessing-openshift)
3. [Manual Installation](#manual-installation)
4. [Installation using the OLM](#installation-using-the-olm)
5. [Deploying NGINX Ingress Operator](#deploying-nginx-ingress-operator)
6. [Deploying TCP service](#deploying-tcp-service)
7. [Monitoring and Testing TCP Service](#monitoring-and-testing-tcp-service)

# NGINX Ingress Controller on OpenShift

## Getting Started

1. Login with XRDP in the ocp-provisioner node. 

![image](https://user-images.githubusercontent.com/4666871/176195040-6483744f-d743-4a7e-b510-8db87e0c8a57.png)

## Accessing Openshift

### Using the CLI

1. Login via the terminal

```other
[cloud-user@ocp-provisioner ~]$ oc login -u f5admin -p f5admin
```

![image](https://user-images.githubusercontent.com/4666871/176195414-148d37dd-a69d-4695-a518-e9e7a8217586.png)
### Using the UI

1. Open a browser with https://console-openshift-console.apps.ocp.f5-udf.com/

2. Use the HTTP auth provider

![Image.png](https://res.craft.do/user/full/d367a179-adcb-7ce8-0b02-ba52d2a7c917/doc/F7779AE5-6B4A-46F8-A1A7-EE43F5EFE81D/94D70987-375B-4086-8155-6C6767057BED_2/Image.png)

3. Login with the credentials **f5admin/f5admin**

![Image.png](https://res.craft.do/user/full/d367a179-adcb-7ce8-0b02-ba52d2a7c917/doc/F7779AE5-6B4A-46F8-A1A7-EE43F5EFE81D/2BA3166A-B301-4F86-80C0-8062F71E2512_2/Image.png)

## Manual Installation

This will deploy the NGINX ingress operator in the `nginx-ingress-operator-system` namespace.

1. Create a new namespace for installing the operator
```other
oc create ns nginx-ingress-operator-system
```

2. Change terminal context to your new namespace

```other
oc config set-context —current —namespace=nginx-ingress-operator-system
```

3. Deploy the Operator and associated resources:
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

4. Check that the Operator is running:

```other
oc get deployments -n nginx-ingress-operator-system

NAME                                        READY   UP-TO-DATE   AVAILABLE   AGE
nginx-ingress-operator-controller-manager   1/1     1            1           15s
```

5. `Openshift` Additional steps:

In order to deploy NGINX Ingress Controller instances into OpenShift environments, a new SCC is required to be created on the cluster which will be used to bind the specific required capabilities to the NGINX Ingress service account(s). To do so, please run the following command (assuming you are logged in with administrator access to the cluster):

```other
oc apply -f main/resources/scc.yaml
```

## Installation using the OLM

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

```other
oc apply -f https://raw.githubusercontent.com/nginxinc/nginx-ingress-helm-operator/main/resources/scc.yaml
```

## Deploying NGINX Ingress Operator

Now to use NGINX Ingress Controller to expose a TCP service, you have to allow global configurations in the NGINX Ingress Deployment. 

1. Change directory back to home

```other
cd ~
```

2. Open the file that contains YAML configuration for NGINX Plus Ingress Controller

```other
nano 1_kubernetes-ingress/deployments/deployment/1_nginx-plus-ingress_custom.yaml
```

3. Uncomment the section that says:

```other
- -global-configuration=$(POD_NAMESPACE)/nginx-configuration
```

4. Then apply this new configuration file to your cluster

```other
oc apply -f 1_kubernetes-ingress/deployments/deployment/1_nginx_plus-ingress_custom.yaml
```

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
```other
oc apply -f nodeport_custom.yaml
```

## Deploying TCP service

1. Create a new namespace for running your TCP service

```other
oc create ns tcp-test
```

2. Set terminal context to your new namespace

```other
oc config set-context —current —namespace=tcp-test
```

3. Apply the configuration for our example service

```other
Oc apply -f https://raw.githubusercontent.com/nginxinc/kubernetes-ingress/main/examples/custom-resources/basic-configuration/cafe.yaml
```

4. Once that is applied you can check your pods and see coffee and tea running

```other
oc get pods
```

 In order to expose a TCP or UDP application, a [GlobalConfiguration](https://docs.nginx.com/nginx-ingress-controller/configuration/global-configuration/globalconfiguration-resource/) resource needs to be implemented as a [Custom Resource](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/). The Ingress controller only needs one GlobalConfiguration resource.

5. View the GlobalConfiguration resource file

```other
nano globalconfigurationresource.yaml
```

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

5. Apply this configuration us

```other
oc apply -f globalconfigurationresource.yaml
```

To expose a TCP service, you need to create a TransportServer resource. This allows you to configure TCP, UDP, and TLS Passthrough load balancing. For TCP and UDP this resource must be used in conjunction with the GlobalConfiguration resource.

6. Explore configuration file for the TransportServer resource

```other
nano ts.yaml
```

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
```other
oc apply -f ts.yaml
```

7. Once the transport server configuration is applied, grab the nginx-ingress pod name to run the required configuration to expose the tcp service:

```other
oc exec -n nginx-ingress nginx-ingress-[unique name] — nginx -T | grep ‘coffee-tcp’ -C 6
```

8. View the ports available to access our service externally:

```other
oc get svc nginx-ingress -n nginx-ingress
```

## Monitoring and Testing TCP Service

1. To view the TCP streams and zones, use the NGINX+ Dashboard:

![image](https://user-images.githubusercontent.com/4666871/176065586-c4062780-5a67-4521-b326-3bc4440792a3.png)
![image](https://user-images.githubusercontent.com/4666871/176065554-64f4b760-619f-4953-9bc6-4871f7b17ea3.png)

2. You can use the bookmark [http://10.1.1.6:30081/](http://10.1.1.6:30081/) to access the service via browser. Each time you open a new incognito browser you will get loadbalanced to a different POD (you see the POD hash changing under Server name:)

![image](https://user-images.githubusercontent.com/4666871/176065508-63bbeac1-778f-438c-84d7-12111978c46a.png)
