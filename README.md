# Spring Boot Tomcat Session Replication on Kubernetes using Hazelcast

Spring Boot helps you to build microservices easily, and it is the most preferred way of building microservices as now. It comes with different options to share data among the running services, such as caching. It also provides session scope data access using built-in Tomcat HTTP sessions. But when you run multiple microservices, it might be a hassle to replicate sessions and share its data through all microservices. 

In this blog post, we will find out how we can replicate sessions through Spring Boot microservices using Hazelcast with minimal configuration changes. We will also display how it runs on Kubernetes environment with a simple application. 

## Requirements

- Apache Maven to build and run the project.
- A containerization software for building containers. We will use [Docker](https://docs.docker.com/install/) in this guide. 
- A Kubernetes environment. We will use local `minikube` environment as k8s for demonstration. 

## Session Replication Sample

This guide contains a basic Spring Boot microservice code sample using [Hazelcast Tomcat Session Manager](https://github.com/hazelcast/hazelcast-tomcat-sessionmanager). You can see the whole project [here](https://github.com/hazelcast-guides/springboot-tomcat-session-replication-on-kubernetes/tree/master/final) and start building your app in the final directory. However, this guide will start from an initial point and build the application step by step. 

### Getting Started

Firstly, you can clone the Git repository below to your local. This repository contains two directories, `/initial` contains the starting project that you will build upon and `/final` contains the finished project you will build.

```
$ git clone https://github.com/hazelcast-guides/springboot-tomcat-session-replication-on-kubernetes.git
$ cd springboot-tomcat-session-replication-on-kubernetes/initial/
```

### Running the Spring Application 

The application in the initial directory is a basic Spring Boot app having 3 endpoints: 

- `/` is the homepage returning “Homepage” string only
- `/put` is the page where key and value is saved to the current session as an attribute.
- `/get` is the page where the values in the current session can be obtained by keys. 

You can run this application using the commands below:

```
$ mvn clean package
$ java -jar target/springboot-tomcat-session-replication-on-kubernetes-0.1.0.jar
```

Now your app is running at `localhost:8080`. You can test it by using the following on another console: 

```
$ curl "localhost:8080"
$ curl --cookie cookies.txt --cookie-jar cookies.txt -s -L "localhost:8080/put?key=myKey&value=hazelcast"
$ curl --cookie cookies.txt --cookie-jar cookies.txt -s -L "localhost:8080/get?key=myKey"
```

As you noticed, we have to use cookies when testing the application since the data is saved on HTTP sessions and we need to access to the same session. The output should be similar to the following:

```
{"value":"hazelcast","podName":null}
```

The value returns as `hazelcast` since we put it in the second command, and `podName` returns `null` because we are not running the application in k8s environment yet. After the testing, you can kill the running application on your console.

### Running the App in a Container

To create the container (Docker) image of the application, we will use [Jib](https://github.com/GoogleContainerTools/jib) tool. It allows to build containers from Java applications without a Docker file or even changing the `pom.xml` file. To build the image, you can run the command below:

```
$ mvn clean compile com.google.cloud.tools:jib-maven-plugin:1.8.0:dockerBuild
```

This command will compile the application, create a Docker image, and register it to your local container registry. Now, we can run the container using the command below:

```
$ docker run -p 5000:8080 springboot-tomcat-session-replication-on-kubernetes:0.1.0
```

This command runs the application and binds the local `5000` port to the `8080` port of the container. Now, we should be able to access the application using the following commands:

```
$ curl "localhost:5000"
$ curl --cookie cookies.txt --cookie-jar cookies.txt -s -L "localhost:5000/put?key=myKey&value=hazelcast"
$ curl --cookie cookies.txt --cookie-jar cookies.txt -s -L "localhost:5000/get?key=myKey"
```

The results will be the same as before. You can kill the running application after testing. Now, we have a container image to deploy on k8s environment.

### Running the App in Kubernetes Environment

To run the app on k8s, we need a running environment. As stated before, we will be using `minikube` for demonstration. After this point, we presume that your k8s environment is running without any issues.

We will use a deployment configuration which builds a service with two pods. Each of these pods will run one container which is built with our application image. You can see our example configuration file (named as `kubernetes.yaml`) in the repository. Please click [here](https://github.com/hazelcast-guides/springboot-tomcat-session-replication-on-kubernetes/blob/master/final/kubernetes.yaml) to download this file. You can see that we are also setting an environment variable named `MY_POD_NAME` to reach the pod name in the application.

After downloading the configuration file, you can deploy the containers using the command below:

```
$ kubectl apply -f kubernetes.yaml
```
Now, we should have a running deployment. You can check if everything is alright by getting the pod list: 

```
$ kubectl get pods
```

It is time to test our running application. But first, we need the the IP address of the running k8s cluster to access it. Since we are using `minikube`, we can get the cluster IP using the command below:

```
$ minikube ip
```

We have the cluster IP address, thus we can test our application:

```
$ curl --cookie cookies.txt --cookie-jar cookies.txt -s -L "http://[CLUSTER-IP]:31000/put?key=myKey&value=hazelcast"
$ while true; do curl --cookie cookies.txt --cookie-jar cookies.txt -s -L [CLUSTER-IP]:31000/get?key=myKey;echo; sleep 2; done
```
The second command makes a request in a loop in order to see the responses from both pods. At some point, you should see a result similar below:

```
{"value":"hazelcast","podName":"hazelcast-tomcatsessionreplication-statefulset-1"}
{"value":null,"podName":"hazelcast-tomcatsessionreplication-statefulset-0"}
```

This means the pod named `hazelcast-tomcatsessionreplication-statefulset-1` got the put request, and stored the value in its session. However, the other pod couldn't get this data and displayed `null` since there are no session replication between pods. To replicate the sessions, we will use Hazelcast in the next step. Now, you can delete the deployment using the following command:

```
$ kubectl delete -f kubernetes.yaml
```

### Session Replication using Hazelcast Tomcat Session Manager

To configure session replication, firstly we will add some dependencies to our `pom.xml` file:

```
<dependency>
    <groupId>com.hazelcast</groupId>
    <artifactId>hazelcast-tomcat85-sessionmanager</artifactId>
    <version>${hazelcast-tomcat-sessionmanager.version}</version>
</dependency>
<dependency>
    <groupId>com.hazelcast</groupId>
    <artifactId>hazelcast</artifactId>
    <version>${hazelcast.version}</version>
</dependency>
<dependency>
    <groupId>com.hazelcast</groupId>
    <artifactId>hazelcast-kubernetes</artifactId>
    <version>${hazelcast-kubernetes.version}</version>
</dependency>
```

The first dependency is for Hazelcast Tomcat Session Manager, the second one is for Hazelcast IMDG itself, and the last one is for Hazelcast's Kubernetes Discovery Plugin which helps Hazelcast members to discover each other on a k8s environment. Now, we just need to add some configuration beans to enable session replication:

```
@Bean
public Config hazelcastConfig() {
    Config config = new Config();
    config.setProperty( "hazelcast.logging.type", "slf4j" );
    config.setInstanceName("hazelcastInstance");
    JoinConfig joinConfig = config.getNetworkConfig().getJoin();
    joinConfig.getMulticastConfig().setEnabled(false);
    joinConfig.getKubernetesConfig().setEnabled(true);
    return config;
}

@Bean
public HazelcastInstance hazelcastInstance(Config hazelcastConfig) {
    return Hazelcast.getOrCreateHazelcastInstance(hazelcastConfig);
}

@Bean
public WebServerFactoryCustomizer<TomcatServletWebServerFactory> customizeTomcat(HazelcastInstance hazelcastInstance) {
    return (factory) -> {
        factory.addContextCustomizers(context -> {
            HazelcastSessionManager manager = new HazelcastSessionManager();
            manager.setSticky(false);
            manager.setHazelcastInstanceName("hazelcastInstance");
            context.setManager(manager);
        });
    };
}
```  

The first bean creates a Hazelcast `Config` object to configure Hazelcast members. We enable the k8s config for discovery. The second bean creates the Hazelcast member using the `hazelcastConfig` bean. The third one customizes Tomcat instance in Spring Boot to use Hazelcast Tomcat Session Manager. Please note that the Hazelcast instance name for `HazelcastSessionManager` object and the instance name for Hazelcast config bean should be the same. Otherwise, `HazelcastSessionManager` wouldn't access the running Hazelcast instance.

Our application with session replication is now ready to go. We do not need to change anything else because we are already using HTTP sessions to store data. 

### Running the App with Tomcat Session Replication in Kubernetes Environment 

Before deploying our updated application on k8s, you should create a `rbac.yaml` file which you can find it [here](https://github.com/hazelcast-guides/springboot-tomcat-session-replication-on-kubernetes/blob/master/final/rbac.yaml). This is the role-based access control (RBAC) configuration which is used to give access to the Kubernetes Master API from pods. Hazelcast requires read access to auto-discover other Hazelcast members and form Hazelcast cluster. After creating/downloading the file, apply it using the command below:

```
$ kubectl apply -f rbac.yaml 
```

Now, we can build our container image again and deploy it on k8s:

```
$ mvn clean compile com.google.cloud.tools:jib-maven-plugin:1.8.0:dockerBuild
$ kubectl apply -f kubernetes.yaml 
```

Our application is running, and it is time to test it again: 

```
$ curl --cookie cookies.txt --cookie-jar cookies.txt -s -L "http://[CLUSTER-IP]:31000/put?key=myKey&value=hazelcast"
$ while true; do curl --cookie cookies.txt --cookie-jar cookies.txt -s -L [CLUSTER-IP]:31000/get?key=myKey;echo; sleep 2; done
```
 
Since we configured session replication, we can see the output similar to the following:

```
{"value":"hazelcast","podName":"hazelcast-tomcatsessionreplication-statefulset-0"}
{"value":"hazelcast","podName":"hazelcast-tomcatsessionreplication-statefulset-1"}
```

Now, both of the pods have the same value and our session data is replicated through microservices!

## Conclusion

In this guide, we first developed a simple microservices application which uses HTTP sessions to store data. The usage is very simple, but the session data is not accessible from all microservices on a k8s environment if the sessions are not replicated. In order to replicate the sessions, we used Hazelcast Tomcat Session Manager. The configuration is easy, just by adding a few beans. Lastly, we succeeded to replicate the sessions among pods which helped us to access the same data from all microservices.