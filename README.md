# Kubernetes workshop

## Pre-requisites

### Install Docker

You can follow the instructions to install `docker` from the [Docker website](https://docs.docker.com/engine/installation/). You can verify the installation is successful with:

```bash
$ docker --version
Docker version 1.12.3, build 6b644ec
```

### Install kubectl

You can follow the instructions to install `kubectl` from the [Kubernetes website](http://kubernetes.io/docs/user-guide/prereqs/). You can verify the installation is successful with:

```bash
$ kubectl version
Client Version: version.Info{Major:"1", Minor:"4", GitVersion:"v1.4.6+e569a27", GitCommit:"e569a27d02001e343cb68086bc06d47804f62af6", GitTreeState:"not a git tree", BuildDate:"2016-11-12T09:26:56Z", GoVersion:"go1.7.3", Compiler:"gc", Platform:"darwin/amd64"}
Server Version: version.Info{Major:"1", Minor:"4", GitVersion:"v1.4.5", GitCommit:"5a0a696437ad35c133c0c8493f7e9d22b0f9b81b", GitTreeState:"clean", BuildDate:"1970-01-01T00:00:00Z", GoVersion:"go1.7.1", Compiler:"gc", Platform:"linux/amd64"}
```

### Install minikube

You can follow the instruction to install `minikube` from the [Minikube website](https://github.com/kubernetes/minikube/releases).

> **Please note** that you have to have kubectl installed before you can run minikube.

If the installation is successful, you can run minikube locally with:

```bash
$ minikube start
Starting local Kubernetes cluster...
Kubectl is now configured to use the cluster.
```

#### Setting up minikube ingress

```bash
$ minikube addons enable ingress
```

And issue a request for the default backend:

```bash
$ curl $(minikube ip)
default backend - 404
$ curl -k https://$(minikube ip)
default backend - 404
```

## Containeraise applications

Clone this repository locally and change directory to `01-static` app

```bash
$ git clone https://github.com/uasabi/kubernetes-workshop
$ cd kubernetes-workshop/apps/01-static
```

You can install the dependencies and run the application with:

```bash
$ yarn install
$ yarn start
yarn start v0.27.5
Server running at http://localhost:4000
```

At this point, you can visit [http://localhost:4000](http://localhost:4000) to
verify that the application is running properly.

Before you can create a container for this application, you have to define what
operating system the container will run and what commands should run when it
starts.
All those operations - and much more - are defined inside a template file called
`Dockerfile`.
Create an empty `Dockerfile` in the current directory.

```bash
$ touch Dockerfile
```

The `Dockerfile` describes step-by-step a list of commands needed to create
a container. This is similar to the build tools you're familiar with, such as
Gradle, grunt or sbt.

```
FROM node:8-alpine
WORKDIR /src
COPY . .
RUN yarn install
RUN yarn test
EXPOSE 4000
CMD yarn start
```

TODO: briefly explain each command

You can build the container with:

```bash
$ docker build -t first-container .
Successfully built 985301b648c5
```

You can verify the image was successfully created with:

```bash
$ docker images | grep first-container
first-container                                     latest              8dad0c3d86b2        7 hours ago         74.2MB
```

You can run the container with:

```bash
$ docker run -p 8080:4000 -d first-container
7145b78dfd2c4563aae4f8a9dc7fdf70fb981c4df2d95105fb9f92dccfefb92b
```

You should be able to visit [http://localhost:8080](http://localhost:8080) and
be greeted by "Hello World".

You can list all running containers with:

```bash
$ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED              STATUS              PORTS                    NAMES
7145b78dfd2c        first-container     "/bin/sh -c 'yarn ..."   About a minute ago   Up About a minute   0.0.0.0:8080->4000/tcp   sad_ritchie
```

You can stop a running container with:

```bash
$ docker stop <container_id>
7145b78dfd2c4563aae4f8a9dc7fdf70fb981c4df2d95105fb9f92dccfefb92b
```

You can inject environment variables in the running container with:

```bash
$ docker run -p 8080:4000 -e NAME=Dan -d first-container
7145b78dfd2c4563aae4f8a9dc7fdf70fb981c4df2d95105fb9f92dccfefb92b
```

You should be able to visit [http://localhost:8080](http://localhost:8080) and
this time be greeted by "Hello Dan".

You can attach to running containers. Usually, this is particularly useful to
debug running application. In your particular case you could inspect the value
of the environment variable `NAME` inside the container.

The first step is to obtain the container id with:

```bash
$ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                    NAMES
82e8b99066ec        first-container     "/bin/sh -c 'yarn ..."   2 seconds ago       Up 1 second         0.0.0.0:8080->4000/tcp   kind_lamarr
```

In this particular case, the container id is `82e8b99066ec`. You can attach to
the running container with:

```bash
$ docker exec -ti 82e8b99066ec /bin/sh
/src #
```

You've now access to the operating system of the running container. You can
check that the environment variable was properly set with:

```bash
/src # echo $NAME
Dan
/src #
```

You should be able to `exit` from the running container.

Sometimes it's useful to share files between your computer and the running
container. Let's create a file in the current directory:

```bash
$ echo "Docker FTW" > hello.txt
```

You can launch *any* container and mount the current directory with:

```bash
$ docker run -ti -v ${PWD}:/app ubuntu /bin/bash
root@990438727c0e:/#
```

You current directory is mounted as `/app`, so go ahead and change directory to
inspect the content of the file:

```bash
root@4c57eec1ef59:/# cd app/
root@4c57eec1ef59:/app# cat hello.txt
Docker FTW
root@4c57eec1ef59:/app#
```

The filesystem is shared with your computer. Any change to the file within the
container will be reflected outside too:

```bash
root@4c57eec1ef59:/app# echo "Yo" >> hello.txt
root@4c57eec1ef59:/app# exit
exit
```

You can inspect the file on your computer with:

```bash
$ cat hello.txt
Docker FTW
Yo
```

## Deploying an application locally

### Starting minikube

Start your Kubernetes cluster locally with minikube:

```bash
$ minikube start
```

Minikube runs a virtual machine with docker and kubernetes (among other things). We must tell our local docker client
to talk to the docker daemon running in the minikube VM, rather than our local docker daemon. The following command will set
your docker host and related variables to make this possible:

```bash
$ eval $(minikube docker-env)
```

If the command was successful, you should be able to see few kubernetes containers running inside the virtual machine from your host:

```bash
$ docker ps --format "{{.ID}}: {{.Image}}"
a6d25657a6e1: gcr.io/google_containers/kube-dnsmasq-amd64:1.4
994ef2f40aaa: gcr.io/google_containers/kubedns-amd64:1.8
be2ce6e2422b: gcr.io/google_containers/exechealthz-amd64:1.2
1716372d439e: gcr.io/google_containers/kubernetes-dashboard-amd64:v1.5.0
4818ea28904a: gcr.io/google_containers/pause-amd64:3.0
ee56c7a18942: gcr.io/google_containers/pause-amd64:3.0
c412afff3115: gcr.io/google-containers/kube-addon-manager:v5.1
0f0f5cbaa917: gcr.io/google_containers/pause-amd64:3.0
```

### Containerise "Hello World", again

Since the docker daemon is running in the virtual machine it doesn't have your image:

```bash
$ docker images | grep first-container
```

So you can rebuild it against minikube's docker daemon with:

```
$ docker build -t first-container .
Successfully built 985301b648c5
```

### Deploy to Kubernetes

A deployment defines what application you want to run (which can consist of multiple containers) and how many replicas you want.

In this particular case we want to run our Node.js container as a single container.

```yaml
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: kube-hello-world
spec:
  replicas: 1
  template:
    metadata:
      labels:
        name: kube-hello-world
    spec:
      containers:
      - name: hello-world-nodejs
        image: first-container
        imagePullPolicy: ifNotPresent
        ports:
          - containerPort: 4000
```

This file is already saved as `deployment.yaml` in the `kube` folder.

You can deploy the application to minikube with:

```bash
$ kubectl create -f kube/deployment.yaml
deployment "kube-hello-world" created
```

If the deployment was successful, you should see a running container:

```bash
$ kubectl get pods
NAME                               READY     STATUS    RESTARTS   AGE
kube-hello-world-3757754181-x1kdu   1/1       Running   0          4s
```

If the deployment wasn't successful and the status is `ErrImagePull` you can inspect the deployment logs with:

```bash
$ kubectl describe pod <kube-hello-world-3757754181-x1kdu>
```

### Define a service

The Node.js application is deployed, but it isn't exposed to the outside world. To expose the application you have to deploy a service. A service is similar to a load balancer, requests to the containers are routed through services.

The repository already has a service. You can find it in `kube/service.yaml`.

```yaml
---
apiVersion: v1
kind: Service
metadata:
  labels:
    name: kube-hello-world-service
  name: kube-hello-world-service
spec:
  ports:
  - name: exposed-port
    port: 80
    targetPort: 4000
  selector:
    name: kube-hello-world
```

You can deploy the service with:

```bash
$ kubectl create -f kube/service.yaml
service "kube-hello-world-service" created
```

You can list the services in kubernetes with:

```bash
$ kubectl get services
NAME                      CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kube-hello-world-service   10.0.0.213   <none>        80/TCP    27s
kubernetes                10.0.0.1     <none>        443/TCP   5h
```

If the deployment was successful, you can login into the cluster and issue an http request:

```bash
$ minikube ssh
Boot2Docker version 1.11.1, build master : 901340f - Fri Jul  1 22:52:19 UTC 2016
Docker version 1.11.1, build 5604cbe
```

You can verify the application is running with

```bash
$ curl <10.0.0.213>
Hello World
```

> Please remember to replace <10.0.0.213> with the right IP address for the
> service

If you can read _"Hello World"_ the deployment was successful.

### Define an ingress

Your application is only available within your cluster, but there isn't an easy way to visit the application from the host. You can expose your application to be consumed by any body using an ingress.

The repository already contains a simple ingress manifest in `kube/ingress.yaml`:

```yaml
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: kube-hello-world-ingress
spec:
  rules:
  - http:
      paths:
      - backend:
          serviceName: kube-hello-world-service
          servicePort: 80
        path: /
```

You can create an ingress with:

```bash
$ kubectl create -f kube/ingress.yaml
ingress "kube-hello-world-ingress" created
```

If the installation was successful, you should be able to visit the virtual machine ip on port 80 and be greeted by _"Hello World"_.

You can find the ip address of your cluster with:

```bash
$ minikube ip
192.168.64.3
```

Visit http://<192.168.64.3>

## Deploying secrets

Your application is likely to have some parameters that are essential to the security of the application - for example API tokens and DB passwords. These should be stored as Kubernetes secrets to enable your application to read them. This process is described in the following guide.

See [official docs](http://kubernetes.io/docs/user-guide/secrets/#creating-a-secret-using-kubectl-create-secret) for more complete documentation; what follows is a very abridged version.

### Generate a strong secret

When generating passwords you should use the following code to ensure you are generating strong passwords.

```bash
$ LC_CTYPE=C tr -dc "[:print:]" < /dev/urandom | head -c 32
```

### Create a kubernetes secret

Create a file called **example-secret.yaml** with the following content below.
In this example, when deployed it will create a kubernetes secret called `my-secret`. Feel free to replace the name `my-secret` with something else:

```yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
type: Opaque
data:
  supersecret: <some_secret_encoded_in_base64>
```

> *Please note* that you shouldn't commit passwords or sensitive information in your
> repository. We suggest you use environment variables to load secrets into the
> yaml file.

Secrets are passed to kubernetes as base64 encoded strings. To encode or decode a base64 string use the following commands:

```bash
$ echo -n "yay it is a secret" | base64
$ echo eWF5IGl0IGlzIGEgc2VjcmV0 | base64 -D
```

Now let's deploy our secret to Kubernetes:

```bash
$ kubectl create --file example-secret.yaml
```

### See and edit the stored secrets

You can retrieve the secret with:

```bash
$ kubectl get secrets
$ kubectl describe secret <my-secret>
```

If you wish to edit secrets already loaded in to Kubernetes you can do so by downloading and reapplying the manifest. You can download the secrets as a Yaml file with:

```bash
$ kubectl get secret <my-secret> -o yaml > example-secrets.yaml
```

You can edit the content of  `example-secrets.yaml`, but remember: values are base64 encoded. If you wish to inspect or add a new entry, you need to decode or encode that value.

Once you're done with the changes, you can reapply all the secrets with:

```bash
$ kubectl apply -f example-secrets.yaml
```

> Please note that it's possible to append a key value pair to an existing secret. You can however download the secret's manifest and reapply the changes as explained above.

### Use the secrets

You can mount secrets into your application using either mounted volumes or by using them as environment variables.

The below example shows a deployment that does both.

<details>
<summary>**This yaml is an example! Please do not copy and paste, just use it as a guide to modify your own deployment.yaml!**</summary>

```yaml
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: kube-hello-world
spec:
  volumes:
    - name: "secretstest"
      secret:
        secretName: mysecret
  containers:
    - image: nginx:1.9.6
      name: awebserver
      volumeMounts:
        - mountPath: "/tmp/mysec"
          name: "secretstest"
      env:
        - name: PASSWORD
          valueFrom:
            secretKeyRef:
                name: my-secret
                key: dbpass
        - name: USERNAME
          valueFrom:
            secretKeyRef:
                name: my-secret
                key: dbuser
        - name: HOST
          valueFrom:
            secretKeyRef:
                name: my-secret
                key: dbhost
```
</details>

For your own `deployment.yaml` file you should have an env section in the appropriate place that looks similar to this. The `name` and `key` fields should be the same:

```yaml
env:
  - name: MYSUPERSECRET
    valueFrom:
      secretKeyRef:
          name: <what you have named your secret>
          key: supersecret
```

Once you've updated your deployment file to set the `MYSUPERSECRET` environment variable using the kubernetes secret, you will need to redeploy it:

```bash
$ kubectl create -f kube/deployment.yaml
```

Now when you navigate to https://tgxu172.notprod.homeoffice.gov.uk/ you should see your secret outputted as part of the message.

### Debug with secrets

Sometimes your app doesn't want to talk to an API or a DB and you've stored the credentials or just the details of that in secret.

The following approaches can be used to validate that your secret is set correctly

```bash
$ kubectl exec -ti my-pod -c my-container -- mysql -h\$DBHOST -u\$DBUSER -p\$DBPASS
## or
$ kubectl exec -ti my-pod -c my-container -- openssl verify /secrets/certificate.pem
## or
$ kubectl exec -ti my-pod -c my-container bash
## and you'll naturally have all the environment variables set and volumes mounted.
## however we recommend against outputing them to the console e.g. echo $DBHOST
## instead if you want to assert a variable is set correctly use
$ [[ -z $DBHOST ]]; echo $?
## if it returns 1 then the variable is set.
```

### Debugging deployments

We suggest the following steps:

#### 1. Check your deployment, replicaset and pods created properly

```bash
$ kubectl get deployments
$ kubectl get rs
$ kubectl get pods
```

#### 2. Investigate potential issues with your pods (this is most likely)

If the get pods command shows that your pods aren't all running then this is likely where the issue is.
You can get further details on why the pods couldn't be deployed by running:

```bash
$ kubectl describe pods *pods_name_here*
```

If your pods are running you can check they are operating as expected by `exec`ing into them (this gets you a shell on one of your containers).

```bash
$ kubectl exec -ti *pods_name_here* -c *container_name_here* bash
```

> **Please note** that the `-c` argument isn't needed if there is only one container in the pod.

You can then try curling your application to see if it is alive and responding as expected. e.g.

```bash
$ curl localhost:4000
```

#### 3. Investigate potential issues with your service

A good way to do this is to run a container in your namespace with a bash terminal:

```bash
$ kubectl run -ti --image busybox debugger bash
```

From this container you can then try curling your service. Your service will have a nice DNS name by default, so you can for example run:

```bash
$ curl my-service-name
```

#### 4. Investigate potential issues with ingress

Minikube runs an ingress service using nginx. It's possible to ssh into the nginx container and cat the `nginx.conf` to inspect the configuration for nginx.

In order to attach to the nginx container, you need to know the name of the container:

```shell
$ kubectl get pods
NAME                               READY     STATUS    RESTARTS   AGE
default-http-backend-2kodr         1/1       Running   1          5d
acp-hello-world-3757754181-x1kdu   1/1       Running   2          6d
ingress-3879072234-5f4uq           1/1       Running   2          5d
```

You can attach to the running container with:

```bash
$ kubectl exec -ti <ingress-3879072234-5f4uq> bash
```

You're inside the container. You can cat the `nginx.conf` with:

```bash
$ cat /etc/nginx/nginx.conf
```

You can also inspect the logs with:

```bash
$ kubectl logs <ingress-3879072234-5f4uq>
```
