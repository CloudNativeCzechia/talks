# Cloud Native Brno Kubernetes 101 Live Demo

The environment was setup through mpolednik's [shellscripts](https://github.com/mpolednik/shellhelpers).

## Demo 1: minikube

The demo is based on [hello-minikube](https://kubernetes.io/docs/tutorials/stateless-application/hello-minikube/) document.
Start up the minikube environment (using hyperkit, darwin hypervisor built for docker).

```bash
$ ./minikube --vm-driver=hyperkit start
```

Next, jump into docker environment of the minikube (implementation detail - we're not using local docker!).

```bash
$ eval $(./minikube docker-env)
```

We have a sample flask application (nothing fancy) in app/. To properly run in Kubernetes, we need to built it as a docker image.

```bash
$ docker build -t app:latest .
```

When the image is built, we may deploy the app to Kubernetes cluster using deployment resource.

```yaml
apiVersion: v1
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    run: app
  name: app
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      run: app
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      labels:
        run: app
    spec:
      containers:
      - image: app:latest
        imagePullPolicy: IfNotPresent
        name: app
        ports:
        - containerPort: 5000
          protocol: TCP
      restartPolicy: Always
      securityContext: {}
```

We may now verify that the deployment and associated pods are running.

```bash
$ kubectl get deployments
NAME      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
app       1         1         1            1           41s

$ kubectl get pods
NAME                  READY     STATUS    RESTARTS   AGE
app-7784f5f5f-8pvvs   1/1       Running   0          1m
```

We cannot access it yet, as it's only running within the cluster. Kubernetes gives us a way to expose an application as a service.

```bash
$ kubectl expose deployment app --type=LoadBalancer
service "app" exposed
```

And minikube has a way to route --type=LoadBalancer services outside of its VM.

```bash
$ ./minikube service app
Opening kubernetes service default/app in default browser...
```

That's it! Time for a cleanup.

```bash
$ eval $(./minikube docker-env -u)

$ ./minikube stop
Stopping local Kubernetes cluster...
```

## Demo 2: Kubernetes "from scratch"

Fire up the VM (VMware Fusion). The VM itself is CentOS 7.5, contains Kubernetes and etcd sources and binaries.

```bash
$ vmrun start "~/Documents/Virtual Machines.localized/CentOS 7 64-bit.vmwarevm/CentOS 7 64-bit.vmx" nogui
```

SSH into the VM.

```bash
$ ssh root@$VM_IP
```

Let's get started with Kubernetes - we first need something to back our resources (etcd).

```bash
$ sh ~/Projects/src/github.com/mpolednik/shellhelpers/start-etcd.sh >/dev/null 2>&1 &
```

We can easily see how etcd works via `etcdctl`. The basic operations are setting a key and retrieving it.

```bash
$ sh ~/Projects/src/github.com/mpolednik/shellhelpers/etcdctl.sh put key ThisIsMyKubernetesResource
OK

$ sh ~/Projects/src/github.com/mpolednik/shellhelpers/etcdctl.sh get key
key
ThisIsMyKubernetesResource
```

But that's a datastore, not a container orchestration. What does `kubectl` say?

```bash
$ sh ~/Projects/src/github.com/mpolednik/shellhelpers/kubectl.sh get nodes
The connection to the server localhost:8080 was refused - did you specify the right host or port?
```

Let's fire up something that gives us proper API to access etcd.

```bash
$ sh ~/Projects/src/github.com/mpolednik/shellhelpers/start-kube-apiserver.sh >/dev/null 2>&1 &
```

That should give us API to interact with resources within Kubernetes.

```bash
$ sh ~/Projects/src/github.com/mpolednik/shellhelpers/kubectl.sh get nodes
No resources found.
```

Progress! We can now use kubectl as with minikube. But does anything happen? There is not a single service (or controller) that knows how to interact with our resources. One of the first controllers we need is a kubelet - the Kubernetes node agent.

```bash
$ sh ~/Projects/src/github.com/mpolednik/shellhelpers/start-kubelet.sh >/dev/null 2>&1 &
```

```bash
$ sh ~/Projects/src/github.com/mpolednik/shellhelpers/kubectl.sh get nodes
NAME                    STATUS    ROLES     AGE       VERSION
localhost.localdomain   Ready     <none>    11s       v1.9.7
```

```yaml
$ sh ~/Projects/src/github.com/mpolednik/shellhelpers/kubectl.sh get nodes localhost.localdomain -o yaml
apiVersion: v1
kind: Node
metadata:
  annotations:
    volumes.kubernetes.io/controller-managed-attach-detach: "true"
  creationTimestamp: 2018-04-23T19:35:13Z
  labels:
    beta.kubernetes.io/arch: amd64
    beta.kubernetes.io/os: linux
    kubernetes.io/hostname: localhost.localdomain
  name: localhost.localdomain
  resourceVersion: "144"
  selfLink: /api/v1/nodes/localhost.localdomain
  uid: 71879cc6-472d-11e8-bfe2-000c2968a91c
spec:
  externalID: localhost.localdomain
status:
  addresses:
  - address: 192.168.40.128
    type: InternalIP
  - address: localhost.localdomain
    type: Hostname
  allocatable:
    cpu: "2"
    memory: 3926680Ki
    pods: "110"
  capacity:
    cpu: "2"
    memory: 4029080Ki
    pods: "110"
  conditions:
  - lastHeartbeatTime: 2018-04-23T19:36:13Z
    lastTransitionTime: 2018-04-23T19:35:13Z
    message: kubelet has sufficient disk space available
    reason: KubeletHasSufficientDisk
    status: "False"
    type: OutOfDisk
  - lastHeartbeatTime: 2018-04-23T19:36:13Z
    lastTransitionTime: 2018-04-23T19:35:13Z
    message: kubelet has sufficient memory available
    reason: KubeletHasSufficientMemory
    status: "False"
    type: MemoryPressure
  - lastHeartbeatTime: 2018-04-23T19:36:13Z
    lastTransitionTime: 2018-04-23T19:35:13Z
    message: kubelet has no disk pressure
    reason: KubeletHasNoDiskPressure
    status: "False"
    type: DiskPressure
  - lastHeartbeatTime: 2018-04-23T19:36:13Z
    lastTransitionTime: 2018-04-23T19:35:23Z
    message: kubelet is posting ready status
    reason: KubeletReady
    status: "True"
    type: Ready
  daemonEndpoints:
    kubeletEndpoint:
      Port: 10250
  images:
  - names:
    - localhost:5000/helloworld@sha256:2ba55cb08f236a70cabddbc64a4bd39e0120af4f3bbd91344f2d3a84309eeba1
    - helloworld:latest
    - localhost:5000/helloworld:latest
    sizeBytes: 438569165
  - names:
    - docker.io/ubuntu@sha256:9ee3b83bcaa383e5e3b657f042f4034c92cdd50c03f73166c145c9ceaea9ba7c
    - docker.io/ubuntu:latest
    sizeBytes: 112925504
  - names:
    - docker.io/registry@sha256:672d519d7fd7bbc7a448d17956ebeefe225d5eb27509d8dc5ce67ecb4a0bce54
    - docker.io/registry:2
    sizeBytes: 33281203
  nodeInfo:
    architecture: amd64
    bootID: 42423acb-add9-4313-8d7b-6b77e4e40230
    containerRuntimeVersion: docker://1.13.1
    kernelVersion: 3.10.0-693.21.1.el7.x86_64
    kubeProxyVersion: v1.9.7
    kubeletVersion: v1.9.7
    machineID: 9f535b15fc2d4a018481133c2bc68311
    operatingSystem: linux
    osImage: CentOS Linux 7 (Core)
    systemUUID: B7194D56-CC26-9A60-C114-C1E16468A91C
```

Node is our first resource. Note that it is created and maintained by Kubelet itself, but may be extended through Kubelet APIs. We may think of node resource as something internal to the cluster - let's get into creating our own resource(s).

First, we need to build our app,

```bash
$ docker build -t helloworld:latest .
```

and make sure that we have a local registry that we can use to store the image.

```bash
$ docker run -d -p 5000:5000 --restart=always --name registry registry:2
```

With that set up, let's push the image to the registry and verify it's there.

```bash
$ docker tag helloworld localhost:5000/helloworld
$ docker push localhost:5000/helloworld
$ docker pull localhost:5000/helloworld
Trying to pull repository localhost:5000/helloworld ...
latest: Pulling from localhost:5000/helloworld
Digest: sha256:2ba55cb08f236a70cabddbc64a4bd39e0120af4f3bbd91344f2d3a84309eeba1
Status: Image is up to date for localhost:5000/helloworld:latest
```

How do we start the image in Kubernetes? `kubectl` can create a resource that defines a deployment.

```bash
$ sh ~/Projects/src/github.com/mpolednik/shellhelpers/kubectl.sh run app --image=localhost:5000/helloworld:latest --replicas=2 --port=80
deployment "app" created
```

```bash
$ sh ~/Projects/src/github.com/mpolednik/shellhelpers/kubectl.sh get deployments
NAME      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
app       2         0         0            0           44s
```

Why CURRENT == 0? kubelet or kube-api-server are not able to handle deployment. There is a separate service able to do that - kube-controller-manager.

```bash
$ sh ~/Projects/src/github.com/mpolednik/shellhelpers/start-kube-controller-manager.sh >/dev/null 2>&1 &
```

```bash
$ sh ~/Projects/src/github.com/mpolednik/shellhelpers/kubectl.sh get deployments
NAME      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
app       2         2         2            0           3m
```

kube-controller-manager was able to process the resource and determine that there are pods to be started.

```bash
$ sh ~/Projects/src/github.com/mpolednik/shellhelpers/kubectl.sh get pods
NAME                   READY     STATUS    RESTARTS   AGE
app-75b6d4cdc6-jzfvd   0/1       Pending   0          44s
app-75b6d4cdc6-nsf8c   0/1       Pending   0          44s
```

Why are they not running then? They are not scheduled - and again, none of our controllers can handle that part. kubelet requires the pod to be scheduled (aka have proper attributes in the resource) before starting the pod.

```bash
$ sh ~/Projects/src/github.com/mpolednik/shellhelpers/start-kube-scheduler.sh >/dev/null 2>&1 &
```

```bash
$ sh ~/Projects/src/github.com/mpolednik/shellhelpers/kubectl.sh get pods
NAME                   READY     STATUS    RESTARTS   AGE
app-75b6d4cdc6-jzfvd   1/1       Running   0          1m
app-75b6d4cdc6-nsf8c   1/1       Running   0          1m
```

Magic. We've been able to transfer the initial deployment resource into a pod resource, and our controllers filled in the details to make it work.

This cluster is not capable of doing networking yet, but we can still poke around the underlying containers to see that our app really works.

```bash
$ docker ps -a
CONTAINER ID        IMAGE                                                                                               COMMAND                  CREATED             STATUS              PORTS                    NAMES
6f0bb826d69c        localhost:5000/helloworld@sha256:2ba55cb08f236a70cabddbc64a4bd39e0120af4f3bbd91344f2d3a84309eeba1   "python app.py"          49 seconds ago      Up 48 seconds                                k8s_app_app-75b6d4cdc6-jzfvd_default_aa8085b6-472e-11e8-bfe2-000c2968a91c_0
6b88f1a0d10a        localhost:5000/helloworld@sha256:2ba55cb08f236a70cabddbc64a4bd39e0120af4f3bbd91344f2d3a84309eeba1   "python app.py"          49 seconds ago      Up 48 seconds                                k8s_app_app-75b6d4cdc6-nsf8c_default_aa80e204-472e-11e8-bfe2-000c2968a91c_0
737170951eb1        gcr.io/google_containers/pause-amd64:3.0                                                            "/pause"                 50 seconds ago      Up 49 seconds                                k8s_POD_app-75b6d4cdc6-jzfvd_default_aa8085b6-472e-11e8-bfe2-000c2968a91c_0
d33d77f96d50        gcr.io/google_containers/pause-amd64:3.0                                                            "/pause"                 50 seconds ago      Up 49 seconds                                k8s_POD_app-75b6d4cdc6-nsf8c_default_aa80e204-472e-11e8-bfe2-000c2968a91c_0
a005f3d5c2f1        registry:2                                                                                          "/entrypoint.sh /e..."   5 hours ago         Up 25 minutes       0.0.0.0:5000->5000/tcp   registry

$ docker exec -it 6f0bb826d69c /bin/bash
root@app-75b6d4cdc6-jzfvd:/app# curl localhost:5000
HI! IM RUNNING IN KUBERNETES!root@app-75b6d4cdc6-jzfvd:/app# exit
```

And that's the story of Kubernetes resources!
