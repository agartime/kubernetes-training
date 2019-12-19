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

# Recovering from a replica problem

Because we don't have a quota in our systems, if we change our replica to, let's say, 200 kubernetes will try to start them, resulting in our minions to fail and our cluster to be unstable. To solve the problem, we change the replica back to one and restart our minions. We're saved.

# Limits

We can make use of limits in order to be safe when we increase our replicas.

```
[kubernetes@kubemaster projects]$ cat deployment-with-1-container.yml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deployment-with-1-container
  labels:
    department: engineering
spec:
  replicas: 200
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
          resources:
            limits:
              memory: "512Mi"
              cpu: "500m" #Half a core
            requests:
              memory: "256Mi"
              cpu: "250m"

```

If the replica is higher than the available resources that we have, we will see that the missing pods will be waiting:

```
[kubernetes@kubemaster projects]$ kubectl get pods -o wide
NAME                                           READY   STATUS    RESTARTS   AGE   IP          NODE          NOMINATED NODE   READINESS GATES
deployment-with-1-container-66d7d764fb-2lbp5   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-66d7d764fb-2zp7r   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-66d7d764fb-46gxl   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-66d7d764fb-49nrt   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-66d7d764fb-4g84b   1/1     Running   0          10m   10.44.0.1   kubeminion1   <none>           <none>
deployment-with-1-container-66d7d764fb-4mxxx   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-66d7d764fb-5brr8   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-66d7d764fb-5gsfl   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-66d7d764fb-5qg5j   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-66d7d764fb-66xdg   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-66d7d764fb-67b5d   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-66d7d764fb-67qh5   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-66d7d764fb-6bpsn   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-66d7d764fb-7lt6h   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-66d7d764fb-86957   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-66d7d764fb-8w6dz   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-66d7d764fb-98t7j   1/1     Running   0          29m   10.47.0.8   kubeminion2   <none>           <none>
deployment-with-1-container-66d7d764fb-bcf2h   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-66d7d764fb-bdqhb   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-66d7d764fb-bk498   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-66d7d764fb-c8d6d   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-66d7d764fb-ck89s   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-66d7d764fb-cwgq2   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-66d7d764fb-dnsv6   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-66d7d764fb-dqrcs   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-66d7d764fb-dzxl2   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-66d7d764fb-f6jm9   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-66d7d764fb-fwlwd   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-66d7d764fb-fx9b8   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-66d7d764fb-gclmn   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-66d7d764fb-gkh97   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-66d7d764fb-gxwq8   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-66d7d764fb-j99kx   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-66d7d764fb-j9ppx   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-66d7d764fb-jt54t   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-66d7d764fb-jtqxv   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-66d7d764fb-jv9qn   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-66d7d764fb-k7s5b   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-66d7d764fb-kj29k   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-66d7d764fb-mq2lc   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-66d7d764fb-nfcl6   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-66d7d764fb-nlzdn   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-66d7d764fb-nqd2z   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-66d7d764fb-q2dzp   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-66d7d764fb-q2wgh   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-66d7d764fb-qw9gv   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-66d7d764fb-rh82q   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-66d7d764fb-rqh56   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-66d7d764fb-rs6m4   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-66d7d764fb-s88t7   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-66d7d764fb-sckmm   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-66d7d764fb-svb7v   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-66d7d764fb-tm2lc   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-66d7d764fb-vbc4p   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-66d7d764fb-vftvg   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-66d7d764fb-vqdzt   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-66d7d764fb-vvd6r   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-66d7d764fb-w28xf   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-66d7d764fb-w5sb5   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-66d7d764fb-xnxbs   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-66d7d764fb-z7g4q   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-66d7d764fb-zj8rz   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-66d7d764fb-zz97t   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-78cdbf7bc5-2jswr   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-78cdbf7bc5-2s5br   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-78cdbf7bc5-4j9jf   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-78cdbf7bc5-4nc9f   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-78cdbf7bc5-4xqpq   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-78cdbf7bc5-5flht   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-78cdbf7bc5-5gpdh   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-78cdbf7bc5-5kzvv   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-78cdbf7bc5-5wlws   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-78cdbf7bc5-7gjxb   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-78cdbf7bc5-7mmg2   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-78cdbf7bc5-85pbw   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-78cdbf7bc5-8h6tv   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-78cdbf7bc5-8swg8   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-78cdbf7bc5-bd7vh   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-78cdbf7bc5-bpwjc   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-78cdbf7bc5-bsjq7   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-78cdbf7bc5-btlnl   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-78cdbf7bc5-cfrqp   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-78cdbf7bc5-cgwdl   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-78cdbf7bc5-cn2ss   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-78cdbf7bc5-dh4x5   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-78cdbf7bc5-dlvbf   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-78cdbf7bc5-dtf78   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-78cdbf7bc5-f6kkd   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-78cdbf7bc5-flnlz   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-78cdbf7bc5-ft6tz   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-78cdbf7bc5-g7n2v   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-78cdbf7bc5-g9djz   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-78cdbf7bc5-ghtq7   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-78cdbf7bc5-gsn9d   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-78cdbf7bc5-hksbb   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-78cdbf7bc5-jn6tw   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-78cdbf7bc5-jwc2s   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-78cdbf7bc5-k8z97   1/1     Running   1          31m   10.44.0.4   kubeminion1   <none>           <none>
deployment-with-1-container-78cdbf7bc5-l6k96   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-78cdbf7bc5-l9dmt   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-78cdbf7bc5-lh8mx   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-78cdbf7bc5-lk6hk   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-78cdbf7bc5-llz8q   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-78cdbf7bc5-lrbgc   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-78cdbf7bc5-m57dn   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-78cdbf7bc5-m9dzm   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-78cdbf7bc5-mcm2f   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-78cdbf7bc5-mhfp2   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-78cdbf7bc5-mz8bq   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-78cdbf7bc5-n4hw2   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-78cdbf7bc5-plrjg   1/1     Running   0          10m   10.47.0.1   kubeminion2   <none>           <none>
deployment-with-1-container-78cdbf7bc5-qhwz7   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-78cdbf7bc5-qqxkg   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-78cdbf7bc5-rffqx   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-78cdbf7bc5-sbmcq   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-78cdbf7bc5-sp9kg   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-78cdbf7bc5-srmbr   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-78cdbf7bc5-t7ctw   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-78cdbf7bc5-vbxz7   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-78cdbf7bc5-wbbzz   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-78cdbf7bc5-wdlzr   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-78cdbf7bc5-whmg5   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-78cdbf7bc5-xctj2   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-78cdbf7bc5-z8psv   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
deployment-with-1-container-78cdbf7bc5-zbxqf   0/1     Pending   0          10m   <none>      <none>        <none>           <none>
```

# Update policy:
You can make kubernetes to automatically upgrade your Pods and apply policies to say how many Pods will update from time to time, how many seconds to be considered as Ready when upgrading or rollbacking, etc.

```
 replicas: 2
  minReadySeconds: 30
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1 #How many pods at the same time
      maxUnavailable: 1 # How many pods allowed to not respond at max

```

