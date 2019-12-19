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
