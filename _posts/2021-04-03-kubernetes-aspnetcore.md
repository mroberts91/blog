---
layout: post
title:  ASP.NET Core on Kubernetes
categories: [dotnet,aspnetcore,k8s,docker]
---

## Containerized .NET Web Apps
No matter what your language/framework/platform of choice is, it is hard to go anywhere in this industry today without hearing people constantly bringing up terms such as Docker, Kubernetes, Helm, Containers, Microservices... . The list of constant buzz words can go on for days and if you are anything like me and just try to dive right into the docs, learning these technologies can seem initially like an daunting task. I found my biggest hurdle, after conceptualizing these concepts, was setting up an application with the proper structure and tooling that would even allow me to run in a container or on a k8s cluster. 

I stumbled across a dotnet project template the other day in my search for an un-related template in this nice list of [Available templates for dotnet new](https://github.com/dotnet/templating/wiki/Available-templates-for-dotnet-new) that caught my eye, [RobBell.AksApi.Template](https://github.com/robbell/dotnet-aks-api-template). After creating and opening my project created from the template my initial thought was "I really wish I had found this when I was first starting my container journey". What made this template great is that the underlying AspNetCore was very simple and did not distract from the sections of the project that you need to identify, such as the chart (Helm Chart) directory and Dockerfile, both of which are going to be used by your tooling to containerize and deploy an application. Another cool addition is the Liveness and Readiness health checks which can save you another trip to the [docs](https://docs.microsoft.com/en-us/aspnet/core/host-and-deploy/health-checks?view=aspnetcore-5.0).

<br />
<p align="center">
  <img alt="Project Structure" src="https://raw.githubusercontent.com/mroberts91/blog/master/images/aks-template-project-structure.png">
</p>

## Testing Local Deployment
I am developing this on Windows and I have found these prerequisites to work well for quick local testing of a kubernetes deployment.

- [docker](https://docs.docker.com/docker-for-windows/install/) - Follow install documentation 
- [minikube](https://minikube.sigs.k8s.io/docs/) - `choco install minikube`
- [helm](https://helm.sh/docs/intro/install/#from-chocolatey-windows) - `choco install kubernetes-helm`

Once the above is installed here are the following steps to containerize and deploy this application to a local kubernetes cluster.

### Creating the image and push to remote repository
```powershell
# In the root directory of the project
docker build -t mrober23/sampleservice:latest .
docker push mrobert23/sampleservice:latest

# Verify image creation
docker images
#REPOSITORY               TAG     IMAGE ID       CREATED              SIZE
#mrober23/sampleservice   latest  73209fc5bac5   About a minute ago   214MB
```

### Starting the minikube cluster
```powershell
# Create minikube cluster
minikube start

😄  minikube v1.15.1 on Microsoft Windows 10 Pro 10.0.19041 Build 19041
✨  Automatically selected the docker driver
👍  Starting control plane node minikube in cluster minikube
🔥  Creating docker container (CPUs=2, Memory=4000MB) ...
🐳  Preparing Kubernetes v1.19.4 on Docker 19.03.13 ...
🔎  Verifying Kubernetes components...
🌟  Enabled addons: storage-provisioner, default-storageclass
🏄  Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default

# Validate k8s
kubectl get all
#NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
#service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   3m41s
```

### Using Helm to deploy
```powershell
# Create sample release
helm upgrade --install sample .\chart\
#Release "sample" does not exist. Installing it now.
#NAME: sample
#LAST DEPLOYED: Sat Apr  3 13:13:54 2021
#NAMESPACE: default
#STATUS: deployed
#REVISION: 1
#TEST SUITE: None
```
**Note:** This template had a slight bug in the Helm Chart `deployment.yaml` template at the time of writing this. So if you receive the following error, is easily resolved. The issue is just 2 extra tabs near the end of the document.

```powershell
Error: unable to build kubernetes objects from release manifest: error validating "": error validating data: [ValidationError(Deployment.spec.template.spec.containers[0].env[1]): unknown field "livenessProbe" in io.k8s.api.core.v1.EnvVar, ValidationError(Deployment.spec.template.spec.containers[0].env[1]): unknown field "readinessProbe" in io.k8s.api.core.v1.EnvVar]
```

The `readinessProbe` and `livenessProbe` blocks must be aligned with the `env` block. Initial project generation had these block contained in the scope of the env block which causes the above error.
```yaml
apiVersion: apps/v1
kind: Deployment
..
spec:
  ..
  template:
    ..
    spec:
      containers:
        - name: {{ .Values.container.name }}
          image: {{ .Values.container.image }}:{{ .Values.container.tag }}
          imagePullPolicy: {{ .Values.container.pullPolicy }}
          ports:
            - containerPort: {{ .Values.container.port }}
          env:
            - name: apphost
              value: {{ .Values.apphost }}
            - name: appenvironment
              value: {{ .Values.environment}}
          readinessProbe:
            httpGet:
              path: /health/ready
              port: {{ .Values.container.port }}
            initialDelaySeconds: {{ .Values.container.probeDelay }}
          livenessProbe:
            httpGet:
              path: /health/live
              port: {{ .Values.container.port }}
            initialDelaySeconds: {{ .Values.container.probeDelay }}
```

### Verifying the Helm Deployment
You can check your helm deployments with the `helm` and `kubectl` cli tools.
```powershell
helm list
#NAME    NAMESPACE REVISION
#sample  default   1

kubectl get deployments
#NAME                READY   UP-TO-DATE   AVAILABLE
#sample-deployment   3/3     3            3        
```

Monitoring the liveness and readiness health checks in action. As you can see the pods are set to state `Running` but each pod is 0/1 Ready. That is because in the above `deployment.yaml` we can see the `initialDelaySeconds` set to our variable which is 30 seconds. After the delay both health checks pass and the service is up and running.
```powershell
kubectl get po
#NAME                                 READY   STATUS    RESTARTS   AGE
#sample-deployment-56d74fd69d-52nnl   0/1     Running   0          28s
#sample-deployment-56d74fd69d-9grw2   0/1     Running   0          28s
#sample-deployment-56d74fd69d-xph47   0/1     Running   0          28s

# Delay has passed
kubectl get po
#NAME                                     READY   STATUS    RESTARTS   AGE
#pod/sample-deployment-56d74fd69d-52nnl   1/1     Running   0          49s
#pod/sample-deployment-56d74fd69d-9grw2   1/1     Running   0          49s
#pod/sample-deployment-56d74fd69d-xph47   1/1     Running   0          49s
```

### Testing the deployment

Exposing the application to test locally with minikube. There is one endpoint that is initially available with the template application, available at `/api/v1/sample`.
```csharp
[Route("api/v1/[controller]")]
[ApiController]
public class SampleController : ControllerBase
{
    private readonly ILogger<SampleController> logger;

    public SampleController(ILogger<SampleController> logger)
    {
        this.logger = logger;
    }

    [HttpGet]
    public string Get()
    {
        logger.LogInformation("Information World");
        logger.LogWarning("Warning World");
        logger.LogCritical("Critical World");
        logger.LogError("Error World");
        return "Hello, World!";
    }
}
```

```powershell
# Get the service name
kubectl get services
#NAME             TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)
#kubernetes       ClusterIP   10.96.0.1        <none>        443/TCP
#sample-service   NodePort    10.109.195.181   <none>        8888:31783 TCP 

# Expose via minikube
minikube service sample-service
|-----------|----------------|-------------|---------------------------|
| NAMESPACE |      NAME      | TARGET PORT |            URL            |
|-----------|----------------|-------------|---------------------------|
| default   | sample-service |        8888 | http://192.168.49.2:31783 |
|-----------|----------------|-------------|---------------------------|
🏃  Starting tunnel for service sample-service.
|-----------|----------------|-------------|------------------------|
| NAMESPACE |      NAME      | TARGET PORT |          URL           |
|-----------|----------------|-------------|------------------------|
| default   | sample-service |             | http://127.0.0.1:55975 |
|-----------|----------------|-------------|------------------------|
🎉  Opening service default/sample-service in default browser...
❗  Because you are using a Docker driver on windows, the terminal needs to be open to run it.
```

Test the endpoint at the expose localhost url with the tool of your choice.
```powershell
curl http://127.0.0.1:55975/api/v1/sample
#Hello, World!
```

We can also validate that our logging in working by checking the logs on all 3 of the pods (instances of the service) until we find the pod that logged out the statements from the above route call.
```powershell
kubectl get po
#NAME                                 READY   STATUS    RESTARTS   AGE
#sample-deployment-56d74fd69d-52nnl   1/1     Running   0          25m
#sample-deployment-56d74fd69d-9grw2   1/1     Running   0          25m
#sample-deployment-56d74fd69d-xph47   1/1     Running   0          25m

# Find the service instance that was hit with the request and check the logs
kubectl logs sample-deployment-56d74fd69d-52nnl
warn: Microsoft.AspNetCore.HttpsPolicy.HttpsRedirectionMiddleware[3]
      Failed to determine the https port for redirect.
warn: SampleService.Controllers.v1.SampleController[0]
      Warning World
crit: SampleService.Controllers.v1.SampleController[0]
      Critical World
fail: SampleService.Controllers.v1.SampleController[0]
      Error World
```

## Summary
This is obviously a very simple example and does not include any substantive security, an ingress controller or any number of additional layers that would be required for an actual production workload. But this template is great for allowing you to work through the core concepts of using docker, kubernetes and helm to deploy a containerized AspNetCore application to a cluster without adding additional complexity and could also be a great jumping off point for the app of your choice.