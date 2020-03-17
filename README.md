# Spring Boot Tomcat Session Replication on Kubernetes using Hazelcast

[Spring Boot](https://spring.io/projects/spring-boot) is a framework that helps to build microservices easily. In the Java ecosystem, it's one of the most preferred way of building microservices as of now. In a microservice architecture, there are different options to share data among the running services: one of them is caching, which Spring Boot leverages. It also provides session scope data access using built-in [Tomcat](http://tomcat.apache.org/) HTTP sessions. But when running multiple microservices, it quickly becomes a hassle to replicate sessions and share their data across microservices. 

In this blog post, we will find out how we can replicate sessions through Spring Boot microservices using Hazelcast with only minimal configuration settings. We will also show how it can be ran on Kubernetes environment through a simple demo application. 

## Requirements

- [Apache Maven](https://maven.apache.org/) to build and run the project.
- A containerization software for building containers. We will use [Docker](https://docs.docker.com/install/) in this guide. 
- A [Kubernetes](https://kubernetes.io/) environment. We will use local `minikube` environment as k8s for demonstration.

## Session Replication Sample

This post contains a basic Spring Boot microservice code sample using [Hazelcast Tomcat Session Manager](https://github.com/hazelcast/hazelcast-tomcat-sessionmanager). You can see the whole project [here](https://github.com/hazelcast-guides/springboot-tomcat-session-replication-on-kubernetes/tree/master/final) and start building your app in the final directory. 

However, we will start from scratch and build the application step-by-step. 

### Getting Started

Firs, clone the Git repository below. It contains two directories, `/initial` contains the starting project that we will build upon, and `/final` contains the finished result.

```
$ git clone https://github.com/hazelcast-guides/springboot-tomcat-session-replication-on-kubernetes.git
$ cd springboot-tomcat-session-replication-on-kubernetes/initial/
```

### Running the Spring Application 

The application in the `initial` directory is a simple Spring Boot application.
It has 3 different endpoints: 

1. `/` is the homepage returning “Homepage” string only
2. `/put` is the page where key and value are saved to the current session as an attribute
3. `/get` is the page where the values in the current session can be obtained by keys 

The application can be ran using the commands below:

```
$ mvn clean package
$ java -jar target/springboot-tomcat-session-replication-on-kubernetes-0.1.0.jar
```

Now the app is running at <http://localhost:8080>. One can test it by using the following commands in another console prompt: 

```
$ curl "localhost:8080"
$ curl --cookie cookies.txt --cookie-jar cookies.txt -s -L "localhost:8080/put?key=myKey&value=hazelcast"
$ curl --cookie cookies.txt --cookie-jar cookies.txt -s -L "localhost:8080/get?key=myKey"
```

Notice we have to use cookies when testing the application since the data is saved in HTTP sessions. Thus, we need to access the same session. The output should be similar to the following:

```
{"value":"hazelcast","podName":null}
```

`value` is set to `hazelcast` since it was put in the second command. `podName` is set to `null` because we are not running the application in a Kubernetes environment... yet. After testing, you can stop the running application.

### Running the containerized application

In order to create the Docker image of the application, we will use the [Jib](https://github.com/GoogleContainerTools/jib) tool, and its related Maven plugin. It allows to build containers from Java applications without a Docker file or even changing the `pom.xml` file. To build the image, run the command below:

```
$ mvn clean compile com.google.cloud.tools:jib-maven-plugin:1.8.0:dockerBuild
```

This command:

1. Compiles the application
2. Creates a Docker image
3. And registers it in the local container registry

Now, let's run the container using the following command:

```
$ docker run -p 5000:8080 springboot-tomcat-session-replication-on-kubernetes:0.1.0
```

This command runs the application and binds the local `5000` port to the `8080` port of the container. Now, we are able to access the application using the following commands:

```
$ curl "localhost:5000"
$ curl --cookie cookies.txt --cookie-jar cookies.txt -s -L "localhost:5000/put?key=myKey&value=hazelcast"
$ curl --cookie cookies.txt --cookie-jar cookies.txt -s -L "localhost:5000/get?key=myKey"
```

The results will be the same as before. Kill the running application after testing. Now, we have a container image to deploy on Kubernetes.

### Running the application on Kubernetes

To run the app on Kubernetes, we need a running environment. As stated before, we will be using `minikube` for that. After this point, we assume that your Kubernetes environment is running.

We will use a deployment configuration which builds a service with two pods. Each of these pods will run one container which is built with our application image. Please see our example configuration file (named as `kubernetes.yaml`) in the repository. Click [here](https://github.com/hazelcast-guides/springboot-tomcat-session-replication-on-kubernetes/blob/master/final/kubernetes.yaml) to download this file. Note that the `MY_POD_NAME` environment variable is set in order to reach the pod name in the application.

After downloading the configuration file, the containers can be deployed like that:

```
$ kubectl apply -f kubernetes.yaml
```

At this point, we should have a running deployment. Let's check if everything is alright by getting the pod list: 

```
$ kubectl get pods
```

It is time to test the running application. But first, the IP address of the running Kubernetes cluster is required to access it. By using `minikube`, we can get the cluster IP using the command below:

```
$ minikube ip
```

We have the cluster IP address, thus we can test our application:

```
$ curl --cookie cookies.txt --cookie-jar cookies.txt -s -L "http://[CLUSTER-IP]:31000/put?key=myKey&value=hazelcast"
$ while true; do curl --cookie cookies.txt --cookie-jar cookies.txt -s -L [CLUSTER-IP]:31000/get?key=myKey;echo; sleep 2; done
```

The second command makes a request in a loop in order to see the responses from both pods. At some point, a result similar to the one below should be displayed:

```
{"value":"hazelcast","podName":"hazelcast-tomcatsessionreplication-statefulset-1"}
{"value":null,"podName":"hazelcast-tomcatsessionreplication-statefulset-0"}
```

This means the pod named `hazelcast-tomcatsessionreplication-statefulset-1` got the put request, and stored the value in its local session storage. However, the other pod couldn't get this data and displayed `null` since there are no session replication between pods. To replicate the sessions, we will use Hazelcast in the next step. Now, delete the deployment using the following command:

```
$ kubectl delete -f kubernetes.yaml
```

### Session Replication using Hazelcast Tomcat Session Manager

To configure session replication, let's first add some dependencies to the `pom.xml` file:

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

The first dependency is for Hazelcast Tomcat Session Manager, the second one is for Hazelcast IMDG itself, and the last one is for Hazelcast's Kubernetes Discovery Plugin. The latter helps Hazelcast members to discover each other on a Kubernetes environment.

At this point, only some configuration beans are necessary to enable session replication:

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

1. The first bean creates a Hazelcast `Config` object to configure Hazelcast members. We enable the Kubernetes configuration for discovery.
2. The second bean creates the Hazelcast member using the `hazelcastConfig` bean.
3. The third one customizes Tomcat instance in Spring Boot to use Hazelcast Tomcat Session Manager.

Please note that the Hazelcast instance name for `HazelcastSessionManager` object and the instance name for Hazelcast config bean should be the same. Otherwise, `HazelcastSessionManager` wouldn't access the running Hazelcast instance.

Our application with session replication is now ready to go. We do not need to change anything else because we are already using HTTP sessions to store data. 

### Running the App with Tomcat Session Replication in Kubernetes Environment 

Before deploying our updated application on Kubernetes, one should create a `rbac.yaml` file which can be found [here](https://github.com/hazelcast-guides/springboot-tomcat-session-replication-on-kubernetes/blob/master/final/rbac.yaml). This is the role-based access control (RBAC) configuration which is used to give access to the Kubernetes Master API from pods. Hazelcast requires read access to auto-discover other Hazelcast members and form Hazelcast cluster. After creating/downloading the file, apply it using the command below:

```
$ kubectl apply -f rbac.yaml 
```

Now, we can build the container image again and deploy it on Kubernetes:

```
$ mvn clean compile com.google.cloud.tools:jib-maven-plugin:1.8.0:dockerBuild
$ kubectl apply -f kubernetes.yaml 
```

The application is running, and it is time to test it again: 

```
$ curl --cookie cookies.txt --cookie-jar cookies.txt -s -L "http://[CLUSTER-IP]:31000/put?key=myKey&value=hazelcast"
$ while true; do curl --cookie cookies.txt --cookie-jar cookies.txt -s -L [CLUSTER-IP]:31000/get?key=myKey;echo; sleep 2; done
```
 
Since session replication is now configured, the output is similar to the following:

```
{"value":"hazelcast","podName":"hazelcast-tomcatsessionreplication-statefulset-0"}
{"value":"hazelcast","podName":"hazelcast-tomcatsessionreplication-statefulset-1"}
```

Both of the pods have the same value and session data is replicated across applications!

## Conclusion

In this post, we first developed a simple microservices application which uses HTTP sessions to store data. The usage is very simple, but the session data is not accessible from all microservices on a Kubernetes environment if the sessions are not replicated. In order to replicate the sessions, we used Hazelcast Tomcat Session Manager. The configuration is just a matter of configuring 3 beans. Finally, we succeeded to replicate the sessions among pods, which helped us to access the same data from all microservices.
