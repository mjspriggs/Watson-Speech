## STT Client Java Web Application
In this tutorial you will build and run a Java Springboot web service that makes use of Watson Speech-to-Text, which is running as a back-end service. You can use the sample application get started writing your own speech application. 

The application provides examples using Watson Speech-to-Text both with:
- **REST interface.** This is used in the application for transcribing audio files in a batch manner. 
- **WebSocket inteface.** This is used for transcribing streams of audio.

To use this tutorial, it is required that you have previously set up an instance of Watson Speech-to-Text in a Kubernetes or OpenShift cluster to which you have access.

### Architecture diagram

![Diagram](architecture.png)
 
### Resources

GitHub Repo: https://github.com/ibm-build-labs/Watson-NLP/tree/main/STTApplication

### Prerequisites
- Docker is installed on your workstation
- Java 17
- Eclipse, if you want to customize the application
- You already have a STT Runtime service running in a k8/OpenShift cluster

As mentioned before this is a java springboot application. Fiegn library is used to make the API call to STT REST Serving. Below is the list of libraries that are used for this application.
```
<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-thymeleaf</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-openfeign</artifactId>
		</dependency>
		<dependency>
		  <groupId>io.github.openfeign</groupId>
		  <artifactId>feign-httpclient</artifactId>
		</dependency>
</dependencies>
```
### Code
Clone the git repository containing our example code. Go to the directory that contains the code used in this tutorial.
```
git clone https://github.com/ibm-build-labs/Watson-NLP
```
```
cd Watson-NLP/STTApplication
```

## Steps to run in localhost

### 1. Build
To build this application please make sure you are in the **STTApplication** directory.

maven wrapper is used here to build and package the application
```
./mvnw clean package
```
A target directory will be created and application will be packaged in jar. e.g **STTApplication-0.0.1-SNAPSHOT.jar**

### 2. Test
- before testing the application please login Kubernetes cluster and expose the stt service endpoint
```
kubectl port-forward svc/install-1-stt-runtime 1080
```
- Set an environment variable for STT service as below. Java application will read the environment variable and make REST calls to that exposed service.
```
export STT_SERVICE_ENDPOINT=127.0.0.1:1080
```
- Run the application
```
java -jar target/STTApplication-0.0.1-SNAPSHOT.jar
```
- The application you will be exposed at port 8080. You can access the application at
```
http://localhost:8080
```

## Run in Kubernetes
### 1. Build an Image.
Here is a simple docker file we used to build a docker image.
```
FROM openjdk:17-jdk-alpine
ARG JAR_FILE=target/*.jar
COPY ${JAR_FILE} app.jar
ENTRYPOINT ["java","-jar","/app.jar"]
```
- Before building the image please execute the below command to package the application
```
./mvnw clean package
```

- Build the image with the below command. I am using ibm container registry to push the image. You can choose a repository on your own.
```
docker build . -t us.icr.io/watson-core-demo/stt-web-application:v1
```
- Push the image to upstream
```
docker push us.icr.io/watson-core-demo/stt-web-application:v1
```
### 2. Deploy
We are creating two Kubernetes resources here, deployment and a service. In deployment.yaml file you need to modify two things
 - Image location
 - Environmental variable STT_SERVICE_ENDPOINT

Here is a sample deployment.yaml file and highlighted the text you might want to replace.
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: stt-web-app
spec:
  selector:
    matchLabels:
      app: stt-web-app
  replicas: 1
  template:
    metadata:
      labels:
        app: stt-web-app
    spec:
      containers:
      - name: stt-web-app
        image: us.icr.io/watson-core-demo/stt-web-application:v1
        resources:
          requests:
            memory: "500m"
            cpu: "100m"
          limits:
            memory: "1Gi"
            cpu: "500m"
        env:
          - name: STT_SERVICE_ENDPOINT
            value: "install-1-stt-runtime:1080"
        ports:
        - containerPort: 8080
```

Get STT service name from the Kubernetes cluster you have deployed your STT Serving. Here is an example. In my case my STT service name is install-1-stt-runtime and port is 1080 for non tls and for tls 1443
```
kubectl get svc 
```
Output
```
NAME                          TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                         AGE
install-1-stt-runtime         NodePort    172.21.206.51    <none>        1080:30280/TCP,1443:32753/TCP   14d
install-1-tts-runtime         NodePort    172.21.199.140   <none>        1080:31439/TCP,1443:30824/TCP   14d
```

Here I am creating a clusterIP service exposing in port 8080, Here is the yaml file
```
apiVersion: v1
kind: Service
metadata:
  name: stt-web-app
spec:
  type: ClusterIP
  selector:
    app: stt-web-app
  ports:
  - port: 8080
    protocol: TCP
    targetPort: 8080
```

deploy kubernetes resource by executing the below command.
```
kubectl apply -f /deployment/
```
### 3. Test 

Check that the pod and service are running.
```
kubectl get pods
```
```
NAME                                           READY   STATUS    RESTARTS   AGE
stt-web-app-64d9df8f49-4fm97                   1/1     Running   0          25h
```
```
kubectl get svc 
```
```
NAME                          TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                         AGE
stt-web-app                   ClusterIP   172.21.238.164   <none>        8080/TCP                        25h
```

To access the app, you need to do a port-forward
```
kubectl port-forward svc/stt-web-app 8080
```

you can access the app at http://localhost:8080
