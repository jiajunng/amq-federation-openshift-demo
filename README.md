# amq-federation-openshift-demo
This demo deploys an AMQ messaging layer which simulates the messaging federation of two different OpenShift clusters using AMQ Interconnect. As it has became more of an usual usecase where one datacenters located away from one another hence there is a need to connect messaging brokers into one logical cluster.
![demo_end_goal](https://user-images.githubusercontent.com/25560159/79966715-6c131a80-84c0-11ea-9409-1049b6ddb37a.png)

For demo sake, OpenShift namespaces will be used to simulate two different clusters.

## Prerequisites/Requirements
* 1x OpenShift cluster (Tested on v4.3)
* Java environment
  * The AMQ clients (producers/consumers) will be implemented using Red Hat Fuse

## Instructions
1. [Deploy AMQ Interconnect in the first cluster](#deploy-amq-interconnect-in-the-first-cluster)
2. [Deploy AMQ Interconnect in the second cluster](#deploy-amq-interconnect-in-the-second-cluster)
3. [Deploy AMQ Broker in the second cluster](#deploy-amq-broker-in-the-second-region)
4. [Attach the broker to the routing layer](#attach-the-broker-to-the-routing-layer)
5. [Run the producer and consumer](#run-the-producer-and-consumer)  

## Deploy AMQ Interconnect in the first cluster
![amq-cluster1](https://user-images.githubusercontent.com/25560159/79744412-de58f300-8338-11ea-8370-3f807b568367.png)

The first cluster will be called *amq-cluster1*. The AMQ Interconnect Operator will deploy two routers which forms a mesh. 

* Create a *amq-cluster1* project
  ```
  $ oc new-project amq-cluster1
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
 
* Install AMQ Certificate Manager Operator to secure the connection between the two routers.
  * Web Console -> Operators -> OperatorHub -> AMQ Certificate Manager -> Install -> Subscribe
    ```
    $ oc get pods -n openshift-operators
    NAME                                       READY   STATUS    RESTARTS   AGE
    cert-manager-controller-657c78fd47-n8lnp   1/1     Running   0          179m
    ```
    
  * Create certificate to link both regions, Web Console -> Operators -> Installed Operators -> Certificate -> Create   Certificate, E.g:
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

  * Extract the certificate for use when deploying the second cluster
  ```
  oc extract secret/cluster2-inter-router-tls
  ```

* Expose AMQPS port so the interconnect routers between the two clusters can communicate with one another.
  * From *amq-cluster1* namespace, navigate to Web Console -> Operators -> Installed Operators -> AMQ Interconnect -> AMQ   Interconnect -> cluster2-router-mesh -> YAML
  * Include *expose: true* for port 5671
  ```
  - port: 5671
    sslProfile: default
    expose: true
  ```

## Deploy AMQ Interconnect in the second cluster
* To create simulate a second cluster, create another namespace, *amq-cluster2*
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

* Expose AMQPS port to allow communciation from the first cluster.
  * From *amq-cluster2* namespace, navigate to Web Console -> Operators -> Installed Operators -> AMQ Interconnect -> AMQ   Interconnect -> cluster2-router-mesh -> YAML
  * Include *expose: true* for port 5671
    ```
    - port: 5671
      sslProfile: default
      expose: true
    ```

** Validate the links between the two clusters has been established
* Navigate to *cluster2-router-mesh-8080* route and enter the following:
  ```
  Address: (default)
  Port: 443
  User Name: guest@cluster2-router-mesh
  Password: (obtain from the Secret generated by Operator)
  ```
  * You can get the password by running: 
    ```$ oc get secret cluster2-router-mesh-users -o=jsonpath={.data.guest} | base64 -d```
    
* You should see the following:
  ![Interconnect](https://user-images.githubusercontent.com/25560159/79947627-da49e400-84a4-11ea-8ccf-d6969d81431d.png)
  On the Interconnect console, it shows that the 4 routers across the regions are connected.

## Deploy AMQ Broker in the second cluster
* From the *amq-cluster2* namespace, Operators -> Installed Operators -> AMQ Broker -> AMQ Broker -> Create Active MQArtemis:
  ```
  apiVersion: broker.amq.io/v2alpha1
  kind: ActiveMQArtemis
  metadata:
    name: broker1
    namespace: amq-cluster2
  spec:
    deploymentPlan:
      size: 1
      image: 'registry.redhat.io/amq7/amq-broker:7.6'
    acceptors:
      - name: amqp
        port: 5672
        protocols: amqp
  ```

## Attach the broker to the routing layer
** Once the broker is created successfully, attach the broker to the routing layer, Web Console -> Operators -> Installed Operators -> AMQ Interconnect -> AMQ Interconnect -> cluster2-router-mesh -> YAML:
   * Add the following into the YAML:
   ```
   spec:
    connectors:
      - name: my-broker
        host: broker1-hdls-svc.amq-cluster2.svc.cluster.local
        port: 5672
        routeContainer: true
    linkRoutes:
      - prefix: test
        direction: in
        connection: my-broker
      - prefix: test
        direction: out
        connection: my-broker
   ```
   The *test* prefix indicates for which addresses the Routing layer should forward messages to the broker. The address *test* matches the address of the Fuse AMQP clients used to produce/consume messages.
  
## Run the producer and consumer 
  You can run both producer and consumer by running:
  ```
  $ mvn
  ```
  
 * You will be able to see the whole flow on the Interconnect visualization:

 ![Final_result](https://user-images.githubusercontent.com/25560159/79955530-81cd1380-84b1-11ea-9eb1-b5644d8bc645.png)

  
