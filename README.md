Kubernetes training
===================

# To execute commands in a Pod:
`kubectl exec -it pod-with-1-container-and-env env`

# How to select a specific container to execute something when there's more than one container in a Pod
`kubectl exec -it -c tomcat pod-with-1-container-and-env env`

# Logs
`kubectl logs <pod_name>`

You don't need to specify a timestamp in your logs. docker does it for us.
`kubectl logs -f --timestamps=true <pod_name>`

# Node Selector

You can tell kubernetes where you want to host a specific pod. (node-selector)
We can use labels for that. Example:

`kubectl label nodes <node_name> <key=value>`
`kubectl label nodes kubeminion1 app=frontend`
`kubectl label nodes kubeminion2 app=backend`

We can check that a node is correctly labeled by using:
`kubectl describe node <node_name>`
`kubectl describe node kubeminion1`

`kubectl apply -f pod-to-node-selector.yml`

```
[kubernetes@kubemaster projects]$ kubectl get pods -o wide
NAME              READY   STATUS    RESTARTS   AGE     IP          NODE          NOMINATED NODE   READINESS GATES
pod-to-backend    1/1     Running   0          2m38s   10.47.0.1   kubeminion2   <none>           <none>
pod-to-frontend   1/1     Running   0          2m38s   10.44.0.1   kubeminion1   <none>           <none>
```

node selector is too strict. If it can't find a label it won't deploy it. (Example with a pod containing a node selector to a label called pakito, that doesn't exist in our cluster).

```
[kubernetes@kubemaster projects]$ kubectl apply -f pod-to-node-selector.yml
pod/pod-to-unknown-node created
pod/pod-to-backend unchanged
pod/pod-to-frontend unchanged
[kubernetes@kubemaster projects]$ kubectl get pods -o wide
NAME                  READY   STATUS    RESTARTS   AGE     IP          NODE          NOMINATED NODE   READINESS GATES
pod-to-backend        1/1     Running   0          9m55s   10.47.0.1   kubeminion2   <none>           <none>
pod-to-frontend       1/1     Running   0          9m55s   10.44.0.1   kubeminion1   <none>           <none>
pod-to-unknown-node   0/1     Pending   0          27s     <none>      <none>        <none>           <none>
[kubernetes@kubemaster projects]$ kubectl describe pod pod-to-unknown-node
Name:         pod-to-unknown-node
Namespace:    default
Priority:     0
Node:         <none>
Labels:       <none>
Annotations:  kubectl.kubernetes.io/last-applied-configuration:
                {"apiVersion":"v1","kind":"Pod","metadata":{"annotations":{},"name":"pod-to-unknown-node","namespace":"default"},"spec":{"containers":[{"i...
Status:       Pending
IP:           
IPs:          <none>
Containers:
  tomcat:
    Image:        tomcat:alpine
    Port:         8080/TCP
    Host Port:    0/TCP
    Environment:  <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-f5tgt (ro)
Conditions:
  Type           Status
  PodScheduled   False 
Volumes:
  default-token-f5tgt:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-f5tgt
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  app=pakito
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type     Reason            Age                From               Message
  ----     ------            ----               ----               -------
  Warning  FailedScheduling  66s (x2 over 66s)  default-scheduler  0/3 nodes are available: 3 node(s) didn't match node selector.

``` 

If we create the label later in the future, the container will start.
```
[kubernetes@kubemaster projects]$ kubectl label nodes kubeminion1 app=pakito --overwrite
node/kubeminion1 labeled
[kubernetes@kubemaster projects]$ kubectl get pods -o wide
NAME                  READY   STATUS              RESTARTS   AGE     IP          NODE          NOMINATED NODE   READINESS GATES
pod-to-backend        1/1     Running             0          15m     10.47.0.1   kubeminion2   <none>           <none>
pod-to-frontend       1/1     Running             0          15m     10.44.0.1   kubeminion1   <none>           <none>
pod-to-unknown-node   0/1     ContainerCreating   0          6m24s   <none>      kubeminion1   <none>           <none>
[kubernetes@kubemaster projects]$ kubectl get pods -o wide
NAME                  READY   STATUS              RESTARTS   AGE     IP          NODE          NOMINATED NODE   READINESS GATES
pod-to-backend        1/1     Running             0          16m     10.47.0.1   kubeminion2   <none>           <none>
pod-to-frontend       1/1     Running             0          16m     10.44.0.1   kubeminion1   <none>           <none>
pod-to-unknown-node   0/1     ContainerCreating   0          6m34s   <none>      kubeminion1   <none>           <none>
[kubernetes@kubemaster projects]$ kubectl get pods -o wide
NAME                  READY   STATUS              RESTARTS   AGE    IP          NODE          NOMINATED NODE   READINESS GATES
pod-to-backend        1/1     Running             0          16m    10.47.0.1   kubeminion2   <none>           <none>
pod-to-frontend       1/1     Running             0          16m    10.44.0.1   kubeminion1   <none>           <none>
pod-to-unknown-node   0/1     ContainerCreating   0          7m7s   <none>      kubeminion1   <none>           <none>
[kubernetes@kubemaster projects]$ kubectl get pods -o wide
NAME                  READY   STATUS    RESTARTS   AGE     IP          NODE          NOMINATED NODE   READINESS GATES
pod-to-backend        1/1     Running   0          17m     10.47.0.1   kubeminion2   <none>           <none>
pod-to-frontend       1/1     Running   0          17m     10.44.0.1   kubeminion1   <none>           <none>
pod-to-unknown-node   1/1     Running   0          7m34s   10.44.0.2   kubeminion1   <none>           <none>
[kubernetes@kubemaster projects]$ vi README.md 
```

# Node Affinity
Affinity is more flexible than node selector. It allow us to create rules to say in which nodes we would prefer to deploy our pods.

There are two types: `requiredDuringSchedulingIgnoredDuringExecution` and `preferredDuringSchedulingIgnoredDuringExecution`. The first one is similar to node selector but much more verbose.

```
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-node-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/os
            operator: In
            values:
            - linux
            - mac
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 1
          preference:
            matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
              - kubeminion2
  containers:
  - name: nginx-with-node-affinity
    image: nginx:latest
``` 

# Replication

Let's create our first deployment using replication.
``` 
[kubernetes@kubemaster projects]$ cat deployment-with-1-container.yml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deployment-with-1-container
  labels:
    department: engineering
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
        - name: tomcat
          image: tomcat:alpine
          ports: 
            - containerPort: 8080
[kubernetes@kubemaster projects]$ kubectl delete pods --all
pod "pod-to-backend" deleted
pod "pod-to-frontend" deleted
pod "pod-to-unknown-node" deleted
pod "pod-with-node-affinity" deleted
[kubernetes@kubemaster projects]$ kubectl get pods
No resources found in default namespace.
[kubernetes@kubemaster projects]$ kubectl get pods ño wide
Error from server (NotFound): pods "ño" not found
Error from server (NotFound): pods "wide" not found
[kubernetes@kubemaster projects]$ kubectl get pods -o wide
No resources found in default namespace.
[kubernetes@kubemaster projects]$ kubectl apply -f deployment-with-1-container.yml 
deployment.apps/deployment-with-1-container created
[kubernetes@kubemaster projects]$ kubectl get pods -o wide
NAME                                           READY   STATUS    RESTARTS   AGE   IP          NODE          NOMINATED NODE   READINESS GATES
deployment-with-1-container-697bfb9dd7-h4s97   1/1     Running   0          5s    10.44.0.1   kubeminion1   <none>           <none>
deployment-with-1-container-697bfb9dd7-jvd5r   1/1     Running   0          5s    10.47.0.1   kubeminion2   <none>           <none>
```

To check our deployments:
```
[kubernetes@kubemaster projects]$ kubectl get deployments
NAME                          READY   UP-TO-DATE   AVAILABLE   AGE
deployment-with-1-container   2/2     2            2           108s
[kubernetes@kubemaster projects]$ 
```

To check our Replica Set:
```
[kubernetes@kubemaster projects]$ kubectl get rs
NAME                                     DESIRED   CURRENT   READY   AGE
deployment-with-1-container-697bfb9dd7   2         2         2       3m7s
```

If we modify our replica to 4:
```
[kubernetes@kubemaster projects]$ kubectl apply -f deployment-with-1-container.yml 
deployment.apps/deployment-with-1-container configured
[kubernetes@kubemaster projects]$ kubectl get pods
+NAME                                           READY   STATUS    RESTARTS   AGE
deployment-with-1-container-697bfb9dd7-h4s97   1/1     Running   0          5m40s
deployment-with-1-container-697bfb9dd7-hrmz4   1/1     Running   0          6s
deployment-with-1-container-697bfb9dd7-jvd5r   1/1     Running   0          5m40s
deployment-with-1-container-697bfb9dd7-qtztk   1/1     Running   0          6s
```

If we switch to 1 replica and apply, it takes care of everything:
```
[kubernetes@kubemaster projects]$ kubectl apply -f deployment-with-1-container.yml 
deployment.apps/deployment-with-1-container configured
[kubernetes@kubemaster projects]$ kubectl get pods
NAME                                           READY   STATUS        RESTARTS   AGE
deployment-with-1-container-697bfb9dd7-h4s97   0/1     Terminating   0          6m43s
deployment-with-1-container-697bfb9dd7-hrmz4   0/1     Terminating   0          69s
deployment-with-1-container-697bfb9dd7-jvd5r   1/1     Running       0          6m43s
deployment-with-1-container-697bfb9dd7-qtztk   0/1     Terminating   0          69s
[kubernetes@kubemaster projects]$ 
[kubernetes@kubemaster projects]$ 
[kubernetes@kubemaster projects]$ 
[kubernetes@kubemaster projects]$ kubectl get pods
NAME                                           READY   STATUS    RESTARTS   AGE
deployment-with-1-container-697bfb9dd7-jvd5r   1/1     Running   0          6m49s
```
