---
title: Kubernetes-imp-commands
description: kubectl imp commands
tags:
  - notes
draft: false
---
# `kubectl` important commands

### Exec command 

###### execute commands in bash of the container
```kubectl exec -it <pod_name> -- bash```

**For multi-container pod:**
```kubectl exec -it <pod_name> -c <container_name> -- bash```


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
