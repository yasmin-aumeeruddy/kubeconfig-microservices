## Configuring a Kubernetes Microservice

### Build and Deploy the Java microservices

The fastest way to work through this guide is to clone the Git repository and use the projects that are provided inside:

`git clone https://github.com/tomjenningss/kubeconfig-microservices.git`

Navigate to the start directory:

`cd kubeconfig-microservices/guide-kubernetes-microprofile-config/start/`

Check out which Kubernetes version is running:

`kubectl version`

The two microservices you will deploy are called **system** and **inventory**. 

The **system** microservice returns the JVM system properties of the running container and it returns the pod’s name in the HTTP header making replicas easy to distinguish from each other. 

The **inventory** microservice adds the properties from the system microservice to the inventory. This demonstrates how communication can be established between pods inside a cluster. To build these applications, navigate to the start directory and run:

`mvn clean package`

When the build succeeds, run the following command to deploy the necessary Kubernetes resources to serve the application:

`kubectl apply -f kubernetes.yaml`

When this command finishes, wait for the pods to be in the **Ready** state. To check if they are in the **Ready** state:

`kubectl get pods`

When the pods are ready, the output shows 1/1 for **READY** and Running for **STATUS**:

```
NAME                                   READY     STATUS    RESTARTS   AGE
system-deployment-6bd97d9bf6-6d2cj     1/1       Running   0          34s
inventory-deployment-645767664f-7gnxf  1/1       Running   0          34s
```

If you see 0/1 **not ready** status, wait and check again as the pods are starting up. This will change to 1/1 and **Running** when your microservices are ready to receive requests.

Now your microservices are deployed and running with the **Ready** status you are ready to send some requests. 

Take a look at which port the nodes are assigned to and take note:

`kubectl get services`

```
NAME                TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
inventory-service   NodePort   172.21.253.135   <none>        9080:32000/TCP   71m
system-service      NodePort   172.21.176.36    <none>        9080:31000/TCP   71m
```
There are two ports specified under the Port(s) collumn for each service and they are shown as {target port}/{node port}/TCP, for example, 9080:32000/TCP where 9080 is the target port and 32000 is the node port. Take note of each node port shown from the command.

Set the **sysPort** and **invPort** variables to the correct node ports for each service:

`sysPort=<port>` 

`invPort=<port>`

Check that they have been set correctly:

`echo $sysPort && echo $invPort`

You should see an output consisting of both node ports **for example**:

sysPort=3100
invPort = 3200

To find the IP addresses required to access the services, use the following command:

`kubectl describe pods`

This command shows the details for both pods. The IP addresses for the nodes that the pods are deployed on are listed in the output. Look for the IP address that is stated next to the label Node: for each pod. For example, in the following case, the IP address would be 10.114.85.172 for the name deployment and 10.114.85.161 for the ping deployment.

Double click the terminal header to make it full screen in relation to the IDE:

Eg: `theia@thedocker-thomasjennin: /home/project/kubeconfig-microservices/guide-kubernetes-microprofile-config/start`

```
Name:           inventory-deployment-7d8f688cc7-svlts
Namespace:      sn-labs-thomasjennin
Priority:       0
Node:           10.114.85.172/10.114.85.172
Start Time:     Wed, 19 Feb 2020 10:39:26 +0000
...
Name:           system-deployment-d47f94ffd-9qccr
Namespace:      sn-labs-thomasjennin
Priority:       0
Node:           10.114.85.172/10.114.85.172
Start Time:     Wed, 19 Feb 2020 10:39:26 +000
```

Like you did with the node ports, set the sysIP and invIP variables to the right IP addresses for the services:

`sysIP=<IP address>`

`invIP=<IP address>`

Check that they have been set correctly:

`echo $sysIP && echo $invIP`

You should see an output consisting of both IP addresses.

## Making requests to the microservices

Next, you'll use **curl** to make an **HTTP GET** request to the 'system' service. The service is secured with a user ID and password that is passed in the request.

`curl -u bob:bobpwd http://$sysIP:$sysPort/system/properties`

You should see a response that will show you the JVM system properties of the running container.

Similarly, use the following curl command to call the inventory service:

`curl -u bob:bobpwd http://$invIP:$invPort/inventory/systems/system-service`

The inventory service will call the system service and store the response data in the inventory service before returning the result.

In this tutorial, you're going to use a Kubernetes ConfigMap to modify the **X-App-Name:** response header. Take a look at their current values by running the following curl command:

`curl -u bob:bobpwd -D - http://$invIP:$invPort/system/properties -o /dev/null`

## Modifying the System Microservice

The system service is hardcoded to have **system** as the app name. To make this configurable, you'll add the **appName** member and code to set the **X-App-Name** to the **SystemResource.java** file. 

Open up the **SystemResource.java** file:

> [File -> Open] /guide-kubernetes-microprofile-config/start/system/src/main/java/system/SystemResource.java

Replace the the **SystemResource.java** with:

```java
package system;

// CDI
import javax.enterprise.context.RequestScoped;
import javax.inject.Inject;
import javax.ws.rs.GET;
// JAX-RS
import javax.ws.rs.Path;
import javax.ws.rs.Produces;
import javax.ws.rs.core.MediaType;
import javax.ws.rs.core.Response;

import org.eclipse.microprofile.config.inject.ConfigProperty;

@RequestScoped
@Path("/properties")
public class SystemResource {

  @Inject
  @ConfigProperty(name = "APP_NAME")
  private String appName;

  @Inject
  @ConfigProperty(name = "HOSTNAME")
  private String hostname;

  @GET
  @Produces(MediaType.APPLICATION_JSON)
  public Response getProperties() {
    return Response.ok(System.getProperties())
      .header("X-Pod-Name", hostname)
      .header("X-App-Name", appName)
      .build();
  }
}
```

These changes use MicroProfile Config and CDI to inject the value of an environment variable called **APP_NAME** into the **appName** member of the **SystemResource** class. MicroProfile Config supports a number of **config sources** from which to receive configuration, including environment variables.

## Modifying the Inventory Microservice

The inventory service is hardcoded to use **bob** and **bobpwd** as the credentials to authenticate against the system service. You’ll make these credentials configurable using a Kubernetes **secret**.

Open up the **SystemClient.java**

>[File -> Open] /guide-kubernetes-microprofile-config/start/inventory/src/main/java/inventory/client/SystemClient.java

and replace the two lines under

**// Basic Auth Credentials** with:

```java

  // Basic Auth Credentials
  @Inject
  @ConfigProperty(name = "SYSTEM_APP_USERNAME")
  private String username;

  @Inject
  @ConfigProperty(name = "SYSTEM_APP_PASSWORD")
  private String password;
```

These changes use MicroProfile Config and CDI to inject the value of the environment variables **SYSTEM_APP_USERNAME** and **SYSTEM_APP_PASSWORD** into the SystemClient class.

## Creating a ConfigMap and Secret

There are several ways to configure an environment variable in a Docker container. You are going to use a Kubernetes ConfigMap and Kubernetes Secret to set these values. These are resources provided by Kubernetes that are used as a way to provide configuration values to your containers. A benefit is that they can be re-used across multiple containers, including being assigned to different environment variables for the different containers.

Create a ConfigMap to configure the application name with the following kubectl command:

`kubectl create configmap sys-app-name --from-literal name=my-system`

This command deploys a ConfigMap named **sys-app-name** to your cluster. It has a key called name with a value of **my-system**. The **--from-literal** flag allows you to specify individual key-value pairs to store in this ConfigMap. Other available options, such as **--from-file** and **--from-env-file**, provide more versatility as to how to configure. Details about these options can be found in the Kubernetes CLI documentation.

Create a Secret to configure the credentials that the inventory service will use to authenticate against system service with the following kubectl command:

`kubectl create secret generic sys-app-credentials --from-literal username=bob --from-literal password=bobpwd`

This command looks very similar to the command to create a ConfigMap, one difference is the word generic. It means that you’re creating a Secret that is **generic**, which means it is not a specialized type of secret. There are different types of secrets, such as secrets to store Docker credentials and secrets to store public/private key pairs.

A Secret is similar to a ConfigMap, except a Secret is used for sensitive information such as credentials. One of the main differences is that you have to explicitly tell kubectl to show you the contents of a Secret. Additionally, when it does show you the information, it only shows you a Base64 encoded version so that a casual onlooker can't accidentally see any sensitive data. Secrets don’t provide any encryption by default, that is something you’ll either need to do yourself or find an alternate option to configure.

## Updating Kubernetes resources

You will now update your Kubernetes deployment to set the environment variables in your containers, based on the values configured in the ConfigMap and Secret. Edit the **kubernetes.yaml** file (located in the **start** directory). This file defines the Kubernetes deployment. Note the **valueFrom** field. This specifies the value of an environment variable, and can be set from various sources. Sources include a ConfigMap, a Secret, and information about the cluster. In this example **configMapKeyRef** sets the key **name** with the value of the ConfigMap **sys-app-name**. Similarly, **secretKeyRef** sets the keys **username** and **password** with values from the Secret **sys-app-credentials**.

To update the resources open the **yaml** file

> [File -> Open] /guide-kubernetes-microprofile-config/start/kubernetes.yaml

and update the file with

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: system-deployment
  labels:
    app: system
spec:
  selector:
    matchLabels:
      app: system
  template:
    metadata:
      labels:
        app: system
    spec:
      containers:
      - name: system-container
        image: tomjenningss/system:1.0-SNAPSHOT
        ports:
        - containerPort: 9080
        # Set the APP_NAME environment variable
        env:
        - name: APP_NAME
          valueFrom:
            configMapKeyRef:
              name: sys-app-name
              key: name
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: inventory-deployment
  labels:
    app: inventory
spec:
  selector:
    matchLabels:
      app: inventory
  template:
    metadata:
      labels:
        app: inventory
    spec:
      containers:
      - name: inventory-container
        image: tomjenningss/inventory:1.0-SNAPSHOT
        ports:
        - containerPort: 9080
        # Set the SYSTEM_APP_USERNAME and SYSTEM_APP_PASSWORD environment variables
        env:
        - name: SYSTEM_APP_USERNAME
          valueFrom:
            secretKeyRef:
              name: sys-app-credentials
              key: username
        - name: SYSTEM_APP_PASSWORD
          valueFrom:
            secretKeyRef:
              name: sys-app-credentials
              key: password
---
apiVersion: v1
kind: Service
metadata:
  name: system-service
spec:
  type: NodePort
  selector:
    app: system
  ports:
  - protocol: TCP
    port: 9080
    targetPort: 9080
---
apiVersion: v1
kind: Service
metadata:
  name: inventory-service
spec:
  type: NodePort
  selector:
    app: inventory
  ports:
  - protocol: TCP
    port: 9080
    targetPort: 9080
```

## Deploying your changes

You now need rebuild and redeploy the applications for your changes to take effect. Rebuild the application using the following commands, making sure you're in the **start** directory:

`mvn clean package`

Now you need to delete your old Kubernetes deployment and update the deployments. 

`kubectl delete -f kubernetes.yaml`

`kubectl apply -f kubernetes.yaml`

Update the port the nodes that are assigned to and take note:

`kubectl get services`

Set the **sysPort** and **invPort** variables to the correct node ports for each service:

`sysPort=<port>` 

`invPort=<port>`

Check that they have been set correctly:

`echo $sysPort && echo $invPort`

Update the IP addresses required to access the services, use the following command:

`kubectl describe pods`


Like you did with the node ports, set the **sysIP** **invIP** variables to the right IP addresses for the services:

`sysIP=<IP address>`

`invIP=<IP address>`

Check that they have been set correctly:

`echo $sysIP && echo $invIP`

You should see an output consisting of both IP addresses.

You should see the following output from the commands:
```
$ kubectl delete -f kubernetes.yaml
deployment.apps "system-deployment" deleted
deployment.apps "inventory-deployment" deleted
service "system-service" deleted
service "inventory-service" deleted
$ kubectl apply -f kubernetes.yaml
deployment.apps/system-deployment created
deployment.apps/inventory-deployment created
service/system-service created
service/inventory-service created
```

Check the status of the pods for the services with:

`kubectl get -pods`

You should eventually see the status of **Ready** for the two services. Press **Ctrl-C** to exit the terminal command.

Call the updated system service and check the headers using the curl command:

`curl -u bob:bobpwd -D - http://$sysIP:$sysPort/system/properties -o /dev/null`

You should see that the response **X-App-Name** header has changed from system to **my-system**.

Verify that inventory service is now using the Kubernetes Secret for the credentials by making the following curl request (This may take several minutes):

`curl http://$invIP:$invPort/inventory/systems/system-service`

If the request fails, check you've configured the Secret correctly.

## Great Work! You're done!
You have used MicroProfile Config to externalize the configuration of two microservices, and then you configured them by creating a ConfigMap and Secret in your Kubernetes cluster.

If you would like to look at the code for these microservices follow the link to the github repository. For more information about the MicroProfile specification used in this tutorial, visit the MicroProfile website with the link below.

[Github repository MicroProfile.io](https://microprofile.io/)
