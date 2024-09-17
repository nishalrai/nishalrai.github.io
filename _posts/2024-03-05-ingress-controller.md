---
title: Introducing Ingress Controllers - An Essential Insights
description: >-
  Get started with Chirpy basics in this comprehensive overview.
  You will learn how to install, configure, and use your first Chirpy-based website, as well as deploy it to a web server.
author: nisharai
date: 2024-03-05 20:55:00 +0800
categories: [Cloud Native, Microservice]
tags: [getting started]
---

Most of us have heard about the ingress controller at least once when Kubernetes was the topic of the talk and I aim to demystify a fundamental understanding of the ingress controller in this post by answering:

- What is an ingress controller?
- Why do we need an ingress controller in the first place?
- How does the ingress controller work?

Before we begin to learn about ingress controllers, it is essential to first grasp the fundamentals to create a framework for a better understanding of the subject. In Kubernetes, the smallest manageable unit is the pod. Think of it as a small box where you can put your applications or containers. But here’s the catch: pods don’t stick around forever. They can be easily removed or replaced using something called a deployment or a manifest file.

```YAML

apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: my-container
    image: nginx:latest
    ports:
    - containerPort: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-container
        image: nginx:latest
        ports:
        - containerPort: 80
```

Now, when we create a pod in the Kubernetes cluster. It gets its own special IP address. This IP address is unique and internal ip within the node of the Kubernetes cluster. But here’s the tricky part: this address can change! So, we can’t always rely on it to stay the same.

That’s where Kubernetes services come into the picture. They’re like the post office for pods. They group together a bunch of pods and give them one fixed address that never changes, making it easy for other pods to find them. This fixed address can be used both inside the cluster (by the Pods) and from outside (the external user).

![kubernetes](/assets/img/images/k8s/info.png){: width="519" height="408" }

So, whether pods are moving around or being replaced, services make sure everything stays connected.

> In Kubernetes, service objects are handled by the control plane, primarily by the kube-api server. When the user fetches the service manifest file to the cluster, the Kubernetes API server processes and creates the service object. This definition is then stored in the Kubernetes datastore called etcd. Thus. Services are not placed inside Pods and when the user makes a service, Kubernetes gives it an IP address and takes care of sending traffic to the right pods linked to that service.

![](/assets/img/images/k8s/cluster-info.png){: width="519" height="408"}

There are different types of services to choose from, depending on how you want to connect to your pods, which includes:
- **ClusterIP**
- **NodePort**
- **LoadBalancer**
- **ExternalName**

Each type has its own way of connecting pods with their own challenges and usage in the Kubernetes cluster. Here, we will only discuss the NodePort, LoadBalancer, and ClusterIP types of Kubernetes service, which are used to manage traffic to applications running within the cluster.


# NodePort:
NodePort-type service assigns a specific port on all nodes (virtual or physical machines) in the Kubernetes cluster. This port falls within the range of 30,000 to 32,767.
Any traffic directed to this port is forwarded to the service. If the port isn’t specified in the configuration file, it’s automatically assigned within the (30,000 to 32,767) range. Users can access the application using a URL in the format.
`http://<node-ip>:<node-port>`

NodePort is primarily used in development or testing environments.

![](/assets/img/images/k8s/nodePort.png){: width="519" height="408"}

The sample manifest file of nodePort service type:
```YAML
apiVersion: v1
kind: Service
metadata:
  name: my-app
spec:
  type: NodePort
  selector:
    component: my-app
  ports:
    - port: 80
      targetPort: 3000
      nodePort: 31000
```

# ClusterIP
ClusterIP is the default service type in Kubernetes. It provides accessibility to associated Pods from other Pods within the cluster but does not allow external access. Connections are made using internal cluster IP addresses and ports.

![](/assets/img/images/k8s/clusterIP.png){: width="519" height="408"}

So why does this such service type exist in the first place, if no user can access the pods externally? Well, on the contrary, ClusterIP is the most used service type in the Kubernetes cluster and will be discussed in the latter section.

The sample manifest file of ClusterIP service type:
```YAML
apiVersion: v1
kind: Service
metadata:
  name: my-app
spec:
  type: ClusterIP
  selector:
    component: my-app
  ports:
    - port: 80
      targetPort: 80
```

# LoadBalancer
LoadBalancer service functions similarly to a ClusterIP service but opens a port on every node in the cluster, allowing external access. This external accessibility is facilitated through a LoadBalancer implementation provided by the cloud provider. Unlike NodePort, LoadBalancer evenly distributes inbound requests across nodes, enhancing scalability and load distribution. However, deploying a LoadBalancer for each exposed service can significantly increase expenses, as each service gets its own IP address.

![](/assets/img/images/k8s/loadbalancer.png){: width="519" height="408"}

The sample of LoadBalancer service type manifest file:
```YAML
apiVersion: v1
kind: Service
metadata:
  name: my-loadbalancer
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: my-app
```

In summary, NodePort provides external access through specific ports on all nodes, LoadBalancer distributes traffic evenly across nodes for scalability, and ClusterIP allows communication between Pods within the cluster without external access. Since each exposed LoadBalancer service gets its own IP address which increases the expenses considering the current landscape of microservice architecture application and NodePort is not reliable enough due to the fact of possibility of change in cluster’s node – if node ip changes, so does the request to the pod also changes:
`http/https://<new-pod-ip>:<nodePort>`

Each NodePort service can have only one port. Even if you specify two blocks of port, the last one will overwrite the rest.

So, you might be wondering, why do I need to know about all this stuff, especially since we’re talking about the ingress controller? Well, the point of understanding the basics of Kubernetes services is to see where they fall short and how the ingress controller swoops in to save the day.


By understanding the limitations of each type of Kubernetes service discussed earlier, we can appreciate how the ingress controller steps in to solve those problems and make our lives a whole lot easier.


# So, what is an ingress controller?
In the context of Kubernetes, controllers are essential components responsible for maintaining the desired state of objects within the cluster. The Ingress controller is responsible for regulating the inbound traffic to the cluster based on the rules defined by the user in the ingress resources.

![](/assets/img/images/k8s/nginxIngress.png){: width="519" height="408"}

Ingress resources are like blueprints that tell the ingress controller how to direct incoming traffic to different services in the cluster. They act as configuration rules specifying entities like which domain names to use, which paths to follow, and which backend services to connect to. Essentially, ingress resources help control how external users can access applications running in the Kubernetes cluster.

## Why do we need ingress controller in the first place?

![](/assets/img/images/k8s/serviceTypes.png){: width="519" height="408"}

In a setup resembling a microservice architecture, instead of exposing individual Pods using nodePort and relying on external load balancers or direct requests through nodePort, we can employ an ingress controller within the Kubernetes (k8s) cluster.
Rather than using an external proxy or load balancer situated outside the k8s cluster, we introduce a proxy as a Pod within the cluster. This proxy Pod, known as the ingress controller, manages traffic based on predefined rules. To facilitate connectivity, it requires a serviceObject associated with an external load balancer.
This configuration simplifies matters, necessitating only a single public IP address and the potential risk of change in the ip address of the nodePort request. When a request is directed to the load balancer, it ultimately reaches the proxy pod, with all routing rules defined within it termed as ingress resources.
While Ingress Resources are native to Kubernetes and supported by the platform, Ingress Controllers are third-party providers such as nginx, HAProxy, Istio, and others.

![](/assets/img/images/k8s/ingressController.png){: width="519" height="408"}

> The ingress controller can be deployed either as a deployment or daemon set. In a deployment, replicas are set in the configuration file, while in daemon sets, Kubernetes ensures at least one ingress controller Pod is running on each node.

To clarify, Ingress, Ingress Rules, and Ingress Resources manage external access to services in the cluster, primarily for HTTP traffic. They handle load balancing, SSL termination, and name-based virtual hosting. Ingress Controllers, on the other hand, are responsible for implementing these rules, typically with a load balancer, to manage traffic within the cluster.


## How is the nginx proxy-server different from the nginx ingress controller?
The primary purpose of typical nginx porxy server and nginx ingress controller is identical to each other, however, unlike proxy servers like HAProxy or nginx, which require reloading after configuration changes, the Nginx Ingress controller doesn’t need reloading. It continuously monitors the service and strives to maintain the desired state defined by the IngressResource. When a user modifies the ingressResource, the ingressController applies those changes. As a result, it operates autonomously without the need for reloading like other proxy servers.
If there’s an error in the configuration, users can view the logs of the nginx ingress controller to pod to find out the root cause or even verify the configuration of the ingress resource.

![](/assets/img/images/k8s/ingressControllerProcess.png){: width="519" height="408"}

The sample manifest file of the ingress controller – creating a Pod, service and an ingress resource for the ingress controller.

```YAML

apiVersion: apps/v1
kind: Deployment
metadata:
  name: dvwa-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      component: dvwa
  template:
    metadata:
      labels:
        component: dvwa
    spec:
      containers:
      - name: dvwa
        imagePullPolicy: IfNotPresent
        image: vulnerables/web-dvwa

        ports:
        - containerPort: 80

---

apiVersion: v1
kind: Service
metadata:
  name: dvwa-service
spec:
  type: ClusterIP
  selector:
    component: dvwa
  ports:
    - port: 80
      targetPort: 80

---

kind: Ingress
metadata:
  name: ingress-service
spec:
  ingressClassName: nginx
  rules:
    - host: "dvwa.ingress.com"
      http:
        paths:
          - pathType: Prefix
            path: "/"
            backend:
              service:
                name: dvwa-service
                port:
                  number: 80
```

Some of the helpful commands to view the logs, ingress resources and pods in the Kubernetes cluster.

```SHELL

kubectl get namespaces
kubectl get pods -n <name-space>
kubectl logs <pod>/<pod-name> -n <name-space>
kubectl get ingress
kubectl describe <ingress-name>
kubectl exec -it <pod>/<pod-name> -n nginx-ingress -- bash
```

# References
- https://kubernetes.io/docs/concepts/services-networking/service/
- https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/
- https://docs.nginx.com/nginx-ingress-controller/overview/design/
