## Configuring a Kubernetes Microservice

### Build and Deploy the Java microservices

To begin, make sure your Kubernetes environment is set up. Once the terminal has finished outputting messages and is ready for input it should be setup. 

To confirm it is ready please run the following command:

`kubectl version`

You should now see the versions of your kubectl client and server. If so, your environment is all set up. If you do not see the version of your Kubernetes server wait a few moments and repeat the previous command until it is shown.

Now you need to navigate into the project directory that has been provided for you. This contains the implementation of the MicroProfile microservices, configuration for the MicroProfile runtime, and Kubernetes configuration.

`cd guide-kubernetes-microprofile-config/start/`

You will notice their is a 'finish' directory. This contains the finished code for this tutorial for reference.

The two microservices you will deploy are called 'system' and 'inventory'. The system microservice returns JVM properties information about the container it is running in. The inventory microservice adds the properties from the system microservice into the inventory. This demonstrates how communication can be achieved between two microservices in separate pod's inside a Kubernetes cluster. To build the applications with Maven, run the following commands one after the other:

`mvn package -pl system`

`mvn package -pl inventory`

Once the services have been built, you need to deploy them to Kubernetes. To do this use the following command:

`kubectl apply -f kubernetes.yaml`

## Making requests to the microservices

Issue the following command to check the status of your microservices:

`kubectl get --watch pods`

If you see 0/1 beside the status **not ready**, wait a little while and check again. This will change to 1/1 and **Running** when your microservices are ready to receive requests.

Now your microservices are deployed and running with the **Ready** status you are ready to send some requests. Press `Ctrl-C` to exit the terminal command. Your pod currently does not have health checks implemented so even though the above command says **Ready** you application may not be ready to receive requests. Adding health checks is beyond the scope of this tutorial but it is something to keep in mind when using Kubernetes.

Firstly check the IP address of your Kubernetes cluster by running the following command:

`minikube ip`

Now you need to set the environment variable IP to the IP address of your Kubernetes cluster by running the following command:

`IP=$(minikube ip)`

Next, you'll use `curl` to make an **HTTP GET** request to the 'system' service. The service is secured with a user id and password that is passed in the request.

`curl -u bob:bobpwd http://$IP:31000/system/properties`

You should see a response that will show you the JVM system properties of the running container.

Similarly, use the following curl command to call the inventory service:

`curl http://$IP:32000/inventory/systems/system-service`

The inventory service will call the system service and store the response data in the inventory service before returning the result.

In this tutorial, you're going to use a Kubernetes ConfigMap to modify the `X-App-Name:` response header. Take a look at their current values by running the following curl command:

`curl -u bob:bobpwd -D - http://$IP:31000/system/properties -o /dev/null`

## Modifying the System Microservice

The system service is hardcoded to have `system` as the app name. To make this configurable, you'll add the `appName` member and code to set the `X-App-Name` to the `SystemResource.java` file. 

Open up the `SystemResource.java` file:

`[File -> Open] /guide-kubernetes-microprofile-config/start/system/src/main/java/system/SystemResource.java` 

Replace the the `SystemResource.java` with:

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

These changes use MicroProfile Config and CDI to inject the value of an environment variable called `APP_NAME` into the `appName` member of the `SystemResource` class. MicroProfile Config supports a number of `config sources` from which to receive configuration, including environment variables.

## Modifying the Inventory Microservice

The inventory service is hardcoded to use `bob` and `bobpwd` as the credentials to authenticate against the system service. You’ll make these credentials configurable using a Kubernetes `secret`.

Open up the `SystemClient.java`

`[File -> Open] /guide-kubernetes-microprofile-config/start/inventory/src/main/java/inventory/client/SystemClient.java`

and replace the two lines under

`// Basic Auth Credentials` with:

```java

  // Basic Auth Credentials
  @Inject
  @ConfigProperty(name = "SYSTEM_APP_USERNAME")
  private String username;

  @Inject
  @ConfigProperty(name = "SYSTEM_APP_PASSWORD")
  private String password;
```

These changes use MicroProfile Config and CDI to inject the value of the environment variables `SYSTEM_APP_USERNAME` and `SYSTEM_APP_PASSWORD` into the SystemClient class.

## Creating a ConfigMap and Secret

There are several ways to configure an environment variable in a Docker container. You are going to use a Kubernetes ConfigMap and Kubernetes Secret to set these values. These are resources provided by Kubernetes that are used as a way to provide configuration values to your containers. A benefit is that they can be re-used across multiple containers, including being assigned to different environment variables for the different containers.

Create a ConfigMap to configure the application name with the following kubectl command:

`kubectl create configmap sys-app-name --from-literal name=my-system`

This command deploys a ConfigMap named `sys-app-name` to your cluster. It has a key called name with a value of `my-system`. The `--from-literal` flag allows you to specify individual key-value pairs to store in this ConfigMap. Other available options, such as `--from-file` and `--from-env-file`, provide more versatility as to how to configure. Details about these options can be found in the Kubernetes CLI documentation.

Create a Secret to configure the credentials that the inventory service will use to authenticate against system service with the following kubectl command:

`kubectl create secret generic sys-app-credentials --from-literal username=bob --from-literal password=bobpwd`

This command looks very similar to the command to create a ConfigMap, one difference is the word generic. It means that you’re creating a Secret that is `generic`, which means it is not a specialized type of secret. There are different types of secrets, such as secrets to store Docker credentials and secrets to store public/private key pairs.

A Secret is similar to a ConfigMap, except a Secret is used for sensitive information such as credentials. One of the main differences is that you have to explicitly tell kubectl to show you the contents of a Secret. Additionally, when it does show you the information, it only shows you a Base64 encoded version so that a casual onlooker can't accidentally see any sensitive data. Secrets don’t provide any encryption by default, that is something you’ll either need to do yourself or find an alternate option to configure.

## Updating Kubernetes resources
