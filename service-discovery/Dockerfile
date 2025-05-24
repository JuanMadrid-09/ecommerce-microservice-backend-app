FROM openjdk:11
#ARG PROJECT_VERSION=0.1.0
RUN mkdir -p /home/app
WORKDIR /home/app
#COPY cloud-config/ .
COPY target/service-discovery-v0.1.0.jar service-discovery.jar
EXPOSE 9296
ENTRYPOINT ["java", "-jar", "service-discovery.jar"]