First, setup a helper/multitool deployment:

```
kubectl run multitool  --image=praqma/network-multitool
```


Setup an nginx deployment with nginx:1.7.9
```
kubectl run nginx  --image=nginx:1.7.9
```

You can also use the `nginx-simple-deployment.yaml` file to create deployments:
```
[demo@kworkhorse exercises]$ kubectl create -f nginx-simple-deployment.yaml 
deployment "nginx" created
[demo@kworkhorse exercises]$ 
```


The contents of `nginx-simple-deployment.yaml` are as follows:
```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```

Check the deployment is created:
```
[demo@kworkhorse exercises]$ kubectl get deployments
NAME      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx     1         1         1            1           4m
[demo@kworkhorse exercises]$
```


Check if the pods are running:
```
[demo@kworkhorse exercises]$ kubectl get pods -o wide
NAME                     READY     STATUS    RESTARTS   AGE       IP            NODE
multitool-69d6b7fc59-66fbf   1/1       Running   0          17m        10.4.128.19   gke-dcn-cluster-35-default-pool-dacbcf6d-c87z
nginx-569477d6d8-tshgn       1/1       Running   0          10m       10.4.128.18   gke-dcn-cluster-35-default-pool-dacbcf6d-c87z
[demo@kworkhorse exercises]$ 
```

Before we move forward, lets see if we can delete a pod, and if it comes to life automatically:
```
[demo@kworkhorse exercises]$ kubectl delete pod nginx-569477d6d8-tshgn 
pod "nginx-569477d6d8-tshgn" deleted
[demo@kworkhorse exercises]$
```

As soon as we delete a pod, a new one is created, satisfying the desired state by the deployment, which is - it needs at least one pod running nginx. So we see that a **new** nginx pod is created (with a new ID):
```
[demo@kworkhorse exercises]$ kubectl get pods -o wide
NAME                         READY     STATUS    RESTARTS   AGE       IP            NODE
multitool-69d6b7fc59-66fbf   1/1       Running   0          4m        10.4.128.19   gke-dcn-cluster-35-default-pool-dacbcf6d-c87z
nginx-569477d6d8-4msf8       1/1       Running   0          5s        10.4.129.12   gke-dcn-cluster-35-default-pool-dacbcf6d-3918
[demo@kworkhorse exercises]$
```


Can we access nginx webserver at port 80 in this pod? We can - only from within the cluster. First we exec into our multitool:
```
[demo@kworkhorse exercises]$ kubectl exec -it multitool-69d6b7fc59-66fbf bash
```

Then curl the IP of the nginx pod:
```
[root@multitool-69d6b7fc59-66fbf /]# curl -s 10.4.128.18 | grep h1
<h1>Welcome to nginx!</h1>
[root@multitool-69d6b7fc59-66fbf /]# 
```

Expose the deployment as a service - type=ClusterIP:
```
[demo@kworkhorse exercises]$ kubectl expose deployment nginx --port 80 --type ClusterIP
service "nginx" exposed
[demo@kworkhorse exercises]$ 
```

Check the service exists:
```
[demo@kworkhorse exercises]$ kubectl get services 
NAME         TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.4.144.1    <none>        443/TCP   17h
nginx        ClusterIP   10.4.158.94   <none>        80/TCP    37s
[demo@kworkhorse exercises]$ 
```

Notice that this service does not have any external IP, nor does it say anything about any other ports except 80/TCP. This means it is not accessible over internet. But we can still access it from within cluster , using the service IP, not the pod IP:
```
[root@multitool-69d6b7fc59-66fbf /]# curl -s 10.4.158.94 | grep h1
<h1>Welcome to nginx!</h1>
[root@multitool-69d6b7fc59-66fbf /]# 
```


This service is still not reachable from outside, so we re-create this service as NodePort.
```
[demo@kworkhorse exercises]$ kubectl get svc
NAME         TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.4.144.1    <none>        443/TCP   17h
nginx        ClusterIP   10.4.158.94   <none>        80/TCP    15m

[demo@kworkhorse exercises]$ kubectl delete svc nginx
service "nginx" deleted

[demo@kworkhorse exercises]$ kubectl expose deployment nginx --port 80 --type NodePort
service "nginx" exposed

[demo@kworkhorse exercises]$ kubectl get svc
NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
kubernetes   ClusterIP   10.4.144.1     <none>        443/TCP        17h
nginx        NodePort    10.4.147.134   <none>        80:31636/TCP   3s
[demo@kworkhorse exercises]$ 
```

Notice that we still don't have an external IP, but we now have an extra port `31636` for this pod. This port is a **NodePort** exposed on the worker nodes. So now, if we know the IP of our nodes, we can access this nginx service from the internet. First we find the public IP of the nodes:
```
[demo@kworkhorse exercises]$ kubectl get nodes -o wide
NAME                                            STATUS    ROLES     AGE       VERSION        EXTERNAL-IP     OS-IMAGE                             KERNEL-VERSION   CONTAINER-RUNTIME
gke-dcn-cluster-35-default-pool-dacbcf6d-3918   Ready     <none>    17h       v1.8.8-gke.0   35.205.22.139   Container-Optimized OS from Google   4.4.111+         docker://17.3.2
gke-dcn-cluster-35-default-pool-dacbcf6d-c87z   Ready     <none>    17h       v1.8.8-gke.0   35.187.90.36    Container-Optimized OS from Google   4.4.111+         docker://17.3.2
[demo@kworkhorse exercises]$ 
```

Even though we have only one pod, we can access any of the node with this port, and it will eventually routed to our pod. So, lets try to access it from our local work computer:
```
[demo@kworkhorse exercises]$ curl -s 35.205.22.139:31636 | grep h1
<h1>Welcome to nginx!</h1>
[demo@kworkhorse exercises]$ 
```

It works!

But, we do not expect the users to know the IP addresses of our worker nodes. It is not a flexible way of doing things. So we recreate the service as type=LoadBalancer. The type LoadBalancer is only available if your k8s cluster is in any of the cloud providers, GCE, AWS, etc.
```
[demo@kworkhorse exercises]$ kubectl get svc
NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
kubernetes   ClusterIP   10.4.144.1     <none>        443/TCP        17h
nginx        NodePort    10.4.147.134   <none>        80:31636/TCP   4m

[demo@kworkhorse exercises]$ kubectl delete svc nginx
service "nginx" deleted

[demo@kworkhorse exercises]$ kubectl expose deployment nginx --port 80 --type LoadBalancer
service "nginx" exposed

[demo@kworkhorse exercises]$ kubectl get svc
NAME         TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
kubernetes   ClusterIP      10.4.144.1     <none>        443/TCP        17h
nginx        LoadBalancer   10.4.156.119   <pending>     80:31354/TCP   5s
[demo@kworkhorse exercises]$ 
```

In few minutes of time the external IP will have some value instead of the word 'pending' . 
```
[demo@kworkhorse exercises]$ kubectl get svc
NAME         TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
kubernetes   ClusterIP      10.4.144.1     <none>        443/TCP        17h
nginx        LoadBalancer   10.4.156.119   35.205.60.29  80:31354/TCP   5s
[demo@kworkhorse exercises]$
```

Now, we can access it without using any special port:
```
[demo@kworkhorse exercises]$ curl -s 35.205.60.29 | grep h1
<h1>Welcome to nginx!</h1>
[demo@kworkhorse exercises]$
```

------

Increase the number of replicas of nginx to 4:
```
[demo@kworkhorse exercises]$ kubectl scale deployment nginx --replicas=4
deployment "nginx" scaled
[demo@kworkhorse exercises]$ 
```

Check the deployment and pods:
```
[demo@kworkhorse exercises]$ kubectl get deployments
NAME        DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
multitool   1         1         1            1           24m
nginx       4         4         4            4           34m

[demo@kworkhorse exercises]$ kubectl get pods
NAME                         READY     STATUS    RESTARTS   AGE
multitool-69d6b7fc59-66fbf   1/1       Running   0          24m
nginx-569477d6d8-4msf8       1/1       Running   0          20m
nginx-569477d6d8-bv77k       1/1       Running   0          34s
nginx-569477d6d8-s6lsn       1/1       Running   0          34s
nginx-569477d6d8-v8srx       1/1       Running   0          35s
[demo@kworkhorse exercises]$ 
```

Notice that the nginx deployment says Desired=4, Current=4, Available=4. And the pods also show the same. There are now 4 nginx pods running.

You can also scale down - to 2:
```
[demo@kworkhorse exercises]$ kubectl scale deployment nginx --replicas=2
deployment "nginx" scaled
[demo@kworkhorse exercises]$ kubectl get pods
NAME                         READY     STATUS        RESTARTS   AGE
multitool-69d6b7fc59-66fbf   1/1       Running       0          25m
nginx-569477d6d8-4msf8       1/1       Running       0          21m
nginx-569477d6d8-bv77k       0/1       Terminating   0          1m
nginx-569477d6d8-s6lsn       0/1       Terminating   0          1m
nginx-569477d6d8-v8srx       1/1       Running       0          2m
[demo@kworkhorse exercises]$
```

Notice that un-necessary pods are killed immediately.

```
[demo@kworkhorse exercises]$ kubectl get pods
NAME                         READY     STATUS    RESTARTS   AGE
multitool-69d6b7fc59-66fbf   1/1       Running   0          26m
nginx-569477d6d8-4msf8       1/1       Running   0          22m
nginx-569477d6d8-v8srx       1/1       Running   0          2m
[demo@kworkhorse exercises]$ 
```

