# amq-interconnect-openshift-demo
This demo deploys an AMQ messaging layer which simulates the messaging federation of two different OpenShift clusters using AMQ Interconnect. It is often an usual case of having more than one datacenters located away from one another and there is a need to connect messaging brokers into one logical cluster.

## Prerequisites/Requirements
* 1x OpenShift cluster (Tested on v4.3)
  * OpenShift namespaces will be used to simulate two different regions
* Java environment
  * The AMQ clients (producers/consumers) will be implemented using Red Hat Fuse

## Instructions
### Deploy AMQ Interconnect in the first region
![amq-cluster1](https://user-images.githubusercontent.com/25560159/79744412-de58f300-8338-11ea-8370-3f807b568367.png)

The first region will be called *amq-cluster1*.The AMQ Interconnect Operator will deploy two nodes (Routers). Then it links them to form a mesh of size two and uses the AMQ Certificate Manager to secure the connection between both.

* Create a *amq-cluster1* project
  ```
  $ oc new-project amq-cluster1
  ```

* Install AMQ Certificate Manager Operator
  * Web Console -> Operators -> OperatorHub -> AMQ Certificate Manager -> Install -> Subscribe
    ```
    $ oc get pods -n openshift-operators
    NAME                                       READY   STATUS    RESTARTS   AGE
    cert-manager-controller-657c78fd47-n8lnp   1/1     Running   0          179m
    ```
* Install AMQ Interconnect Operator
  * Web Console -> Operators -> OperatorHub -> AMQ Interconnect -> Install
    ```
    $ oc get pods -n amq-cluster1
    NAME                                     READY   STATUS    RESTARTS   AGE
    interconnect-operator-5c8f464bc4-48n7h   1/1     Running   0          92m
    ```
* Deploy the AMQ Interconnect Routing layer
  * From the *amq-cluster1* namespace, Operators -> Installed Operators -> AMQ Interconnect -> AMQ Interconnect -> Create Interconnect
  * You can use the default setting, I have changed the name to *cluster1-router-mesh* so it is easier to identify:
    ```
    apiVersion: interconnectedcloud.github.io/v1alpha1
    kind: Interconnect
    metadata:
      name: cluster1-router-mesh
      namespace: amq-cluster1
    spec:
      deploymentPlan:
        size: 2
        role: interior
        placement: Any
    ```
  * End result should be as follows:
    ```
    $ oc get pods -n amq-cluster1
    NAME                                     READY   STATUS    RESTARTS   AGE
    cluster1-router-mesh-688bfdc477-5jnvp    1/1     Running   0          79m
    cluster1-router-mesh-688bfdc477-f2lsz    1/1     Running   0          79m
    interconnect-operator-5c8f464bc4-48n7h   1/1     Running   0          92m
    ``` 
* Using the AMQ Certificate Manager, create certificate to link both regions, Web Console -> Operators -> Installed Operators -> Certificate -> Create Certificate, E.g:
```
apiVersion: certmanager.k8s.io/v1alpha1
kind: Certificate
metadata:
  name: cluster2-inter-router-tls
spec:
  commonName: cluster1-router-mesh-myproject.cluster2.openshift.com
  issuerRef:
    name: cluster1-router-mesh-inter-router-ca
  secretName: cluster2-inter-router-tls
```
* Extract the certificate for use when deploying the second region
```
oc extract secret/cluster2-inter-router-tls
```
* Expose AMQPS port
  * From *amq-cluster1* namespace, navigate to Web Console -> Operators -> Installed Operators -> AMQ Interconnect -> AMQ   Interconnect -> cluster2-router-mesh -> YAML
  * Include *expose: true* for port 5671
  ```
  - port: 5671
    sslProfile: default
    expose: true
  ```

### Deploy AMQ Interconnect in the second region
* To create simulate a second region, create another namespace, *amq-cluster2*
  ```
  $ oc new-project amq-cluster2
  ```
* Create secret from the previous cluster's certificate:
  ```
  $ oc create secret generic cluster2-inter-router-tls
  ```
* Install AMQ's Interconnect Operator
  * Web Console -> Operators -> OperatorHub -> AMQ Interconnect -> Install
  
* Deploy the AMQ Interconnect Routing layer
  * From the *amq-cluster2* namespace, Operators -> Installed Operators -> AMQ Interconnect -> AMQ Interconnect -> Create Interconnect
  * The YAML should include the router connector to connect to the first region
  ```
  apiVersion: interconnectedcloud.github.io/v1alpha1
  kind: Interconnect
  metadata:
    name: cluster2-router-mesh
    namespace: amq-cluster2
  spec:
    deploymentPlan:
      size: 2
      role: interior
      placement: Any
    sslProfiles:
    - name: inter-cluster-tls
      credentials: cluster2-inter-router-tls
      caCert: cluster2-inter-router-tls
    interRouterConnectors:
    - host: [HERE THE URL TO CLUSTER-1 PORT 55671]
      port: 443
      verifyHostname: false
      sslProfile: inter-cluster-tls

  ```
  Note that the route can be retrieved by running: ``` $ oc get routes -n amq-cluster1 ```

* Expose AMQPS port
  * From *amq-cluster2* namespace, navigate to Web Console -> Operators -> Installed Operators -> AMQ Interconnect -> AMQ   Interconnect -> cluster2-router-mesh -> YAML
  * Include *expose: true* for port 5671
  ```
  - port: 5671
    sslProfile: default
    expose: true
  ```
  
