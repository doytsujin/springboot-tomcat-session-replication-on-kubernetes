FROM openjdk:8-jdk-alpine

VOLUME /tmp

ARG JAR_FILE=target/springboot-tomcat-session-replication-on-kubernetes-0.1.0.jar

ADD ${JAR_FILE} hazelcast-tomcatsessionreplication-demo.jar

ENTRYPOINT ["java","-Djava.security.egd=file:/dev/./urandom","-jar","/hazelcast-tomcatsessionreplication-demo.jar"]
