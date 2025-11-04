---
title: Kubernetes-imp-commands
description: kubectl imp commands
tags:
  - notes
draft: false
---
# `kubectl` important commands

### Cluster information


To get all the cluster information you can use the `get-contexts` command. The `*` tells which context is currently being used.

1. `kubectl config get-contexts`

It is used to see all the cluster and context that the kubectl can see using the kube config file.

```
kubectl config get-contexts                                                                                                                                                                                                          

CURRENT   NAME                 CLUSTER              AUTHINFO                   NAMESPACE

*         do-blr1-k8-dp-labs   do-blr1-k8-dp-labs   do-blr1-k8-dp-labs-admin   

          docker-desktop       docker-desktop       docker-desktop             

          kind-argo-demo       kind-argo-demo       kind-argo-demo             argocd

          kind-kind-cluster    kind-kind-cluster    kind-kind-cluster          

          minikube             minikube             minikube                   default

          rancher-desktop      rancher-desktop      rancher-desktop
          
```

2. `kubectl cluster-info`

This command helps you see the cluster information, like where is the control plane running, coreDNS.
```
kubectl cluster-info

Kubernetes control plane is running at https://52359178-3691-413a-a2da-e4fa937b5ded.k8s.ondigitalocean.com

CoreDNS is running at https://52359178-3691-413a-a2da-e4fa937b5ded.k8s.ondigitalocean.com/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
```


3. `kubectl config use-context <context-name>`

To switch to a different context , you can use use-context command.

#### switch to a different context using `use-context`
```
kubectl config use-context docker-desktop                                                                                                                        Switched to context "docker-desktop"

```

4.  `kubectl exec -it <pod_name> -- bash`
To open the bash shell inside the pod you can use the exec command with the interactive option enabled.

#### Exec command 

###### execute commands in bash of the container
```kubectl exec -it <pod_name> -- bash```

**For multi-container pod:**
If you want to exec commands in bash of specific container, then you can specify the specific container using the `-c` option and passing the `container_name`.
```kubectl exec -it <pod_name> -c <container_name> -- bash```


5. `kubectl explain <api-resource>`

#### Check the K8 CLI documentation
To see the api-resource documentation on CLI we can use the explain command. 
`kubectl explain <api-resource>`

examples:
1. `kubectl explain pod`
2. `kubectl explain pod.spec`
3. `kubectl explain pod.spec.containers`





# Knowledge Nuggets
### ENTRYPOINT, CMD  in docker vs command , args in k8
In docker, if you want to run something on the container startup then you have the option to define it in the Dockerfile which is used to generate the docker image.

The two `keywords` are `ENTRYPOINT` and `CMD`. 
**`ENTRYPOINT`** is always executed whenever the container is spinned up and after that the arguments defined in the **`CMD`** keyword  are passed to the ENTRYPOINT.

Take a simple dockerfile example, if you build a docker image from it and create a docker container from that image.
```
FROM busybox:latest
ENTRYPOINT ["/bin/echo"]
CMD ["hello","world!"]
```

the container on startup would first run the `/bin/echo`  and then associated with it, it would pass the arguments `hello` , `world!` and it could print  `"hello world!"` and then exit.


but while creating the container if you pass argument on the CLI
e.g `docker run <image_name> <your argument>`
then it replaces the CMD values with the provided argument.

The equivalent K8 manifest for this would look like below, where the command is basically the entrypoint and the args are what supplied to the command at container startup.
```
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
    - name: nginx
      image: nginx
      command: ['/bin/echo']
      args: ['hello', 'world!']
```

you can put the above yaml in test.yaml and  test this manifest by doing the dry run 
```
kubectl apply -f test.yaml --dry-run=client        
```
you should see something like 
```
pod/nginx-pod created (dry run)
```


Applying the manifest and then checking the logs shows it produced the similar output to docker container.
```
kubectl apply -f test.yaml
pod/nginx-pod created


kubectl logs nginx-pod             
hello world!
```


To create the container with command and args, you can also do it directly using the kubectl command.
`kubectl run nginx --image=nginx --command -- /bin/echo hello world!`

 it would produce the similar result, here's the general syntax.
`--command <command> <args>`



### In k8 manifest file , you can also write the `command` in the list format
example:
```
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
    - name: nginx
      image: nginx
      command:
      - 'bin/echo'
      - 'hello'
      - 'world!'
```




### Docker EXPOSE intruction

The `Expose` instruction informs docker that the `container` listens on the specified network `port` at runtime.

`Expose` doesn't actually publish the port

It acts more as a documentation between the person who wrote the docker image and the person running the container, about which ports are intended to be published.




