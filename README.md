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
