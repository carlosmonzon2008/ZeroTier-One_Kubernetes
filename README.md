# ZeroTier-One_Kubernetes
## This is an example deployment of ZeroTier One on Kubernetes.

### How to deploy to a Kubernetes cluster, from a terminal that has "kubectl" configured with that cluster.
```console
user@computer:~$ kubectl apply -f ZeroTier_Kubernetes_Deploy.yaml
persistentvolumeclaim/zerotier created
configmap/zerotier-networks created
deployment.apps/zerotier created
```

> ### - In this example I use a ConfigMap to store the \<`network ID`\>, which in case of being more than one must be separated by space. I have also commented the option of a Secret. In case you want to use the Secret is only to comment the ConfigMap part and uncomment the Secret part, and also comment the "configMapKeyRef" block and uncomment the "secretKeyRef" block inside the Deployment.

> In case of using the ConfigMap.
```yaml
---
kind: ConfigMap
apiVersion: v1
metadata:
  name: zerotier-networks
  labels:
    app: zerotier
    app.kubernetes.io/component: zerotier
    app.kubernetes.io/instance: zerotier
    app.kubernetes.io/name: zerotier
    app.kubernetes.io/part-of: zerotier
data:
  NETWORK_IDS: 0123456789abcdef
```
`Replace the value of NETWORK_IDS with the ID(s) of your ZeroTier network(s).`

> In case of using the Secret.
```yaml
---
kind: Secret
apiVersion: v1
metadata:
  name: zerotier-networks
  labels:
    app: zerotier
    app.kubernetes.io/component: zerotier
    app.kubernetes.io/instance: zerotier
    app.kubernetes.io/name: zerotier
    app.kubernetes.io/part-of: zerotier
data:
  NETWORK_IDS: MDEyMzQ1Njc4OWFiY2RlZg==
type: Opaque
```
`Replace the NETWORK_IDS value with the ID(s) of your ZeroTier network(s), base 64 encoded.`

> Example of how to encode in base64 from a Linux terminal.
```console
user@computer:~$ printf 0123456789abcdef | base64
MDEyMzQ1Njc4OWFiY2RlZg==
```


> ### -It is important to keep in mind that for this deployment to work correctly, you must have administrator permissions in the Kubernetes cluster, or at least the necessary permissions to be able to assign "privileged=true" in the container's securityContext, in addition to the "NET_ADMIN" and "SYS_ADMIN" capabilities.
```yaml
---
kind: Deployment
.
.
.
          securityContext:
            privileged: true
            capabilities:
              add:
              - NET_ADMIN
              - SYS_ADMIN
.
.
.
```

> ### - In the Deployment, the block that is enclosed by `##-----------------------------------------------------Optional-----------------------------------------------------`, is to include a "hello world" made with an NGINX container custom image, that when doing a cURL to the IP of the POD from another member peer within the same ZeroTier network, it responds with this:
```console
c:\>curl 10.144.200.248
<html>
    <head>
        <title>Hello, ZeroTier!</title>
    </head>
    <body>
        <h2>Hello, ZeroTier!</h2>
    </body>
</html>
```
`The IP "10.144.200.248" is the IP that was assigned to my POD by my ZeroTier network in my tests. The IP in your case may be different. In this example the cURL was done from a Windows machine connected to the same ZeroTier network of this POD.`

> ### - This is an example of the command to to view the logs from the container "kubernetes-zerotier" inside of the POD.
```console
user@computer:~$ kubectl logs -f -l "app=zerotier,deployment=zerotier" -c kubernetes-zerotier
=> Configuring networks to join
=> Joining networks: [0123456789abcdef]
===> Configuring join: [0123456789abcdef]
=> Starting ZeroTier
===> ZeroTier hasn't started, waiting a second
=> Writing healthcheck for networks: [0123456789abcdef]
=> zerotier-cli info: [200 info 72103ab4e8 1.10.2 OFFLINE]
=> Sleeping infinitely
```
`Although at startup it indicates OFFLINE, after this member's access is approved within the ZeroTier Web portal (https://my.zerotier.com/network/), when accessing within the POD and executing the command "zerotier-cli info" it will appear ONLINE.`

> ### - Commands for getting inside the POD to check the connection status and view the assigned IP address from ZeroTier network.
```console
user@computer:~$ kubectl exec -it $(kubectl get pods -l "app=zerotier,deployment=zerotier" -o name) -c kubernetes-zerotier -- /bin/bash

root@zerotier-5cf9d447c-zhf14:/#

root@zerotier-5cf9d447c-zhf14:/# zerotier-cli info
200 info 72103ab4e8 1.10.2 ONLINE

root@zerotier-5cf9d447c-zhf14:/# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
2: sit0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN group default qlen 1000
    link/sit 0.0.0.0 brd 0.0.0.0
4: eth0@if16: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether ea:97:c8:0b:bc:eb brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.244.0.22/16 brd 10.244.255.255 scope global eth0
       valid_lft forever preferred_lft forever
5: ztc12bw2fe: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 2800 qdisc pfifo_fast state UNKNOWN group default qlen 1000
    link/ether e7:a4:4d:1d:17:d8 brd ff:ff:ff:ff:ff:ff
    inet 10.144.200.248/16 brd 10.144.255.255 scope global ztc12bw2fe
       valid_lft forever preferred_lft forever
```
`Here you can see the network interface that ZeroTier adds to the POD, which in this test is "ztc12bw2fe", and the associated IP "10.144.200.248/16", which is a range belonging to the ZeroTier network to which the POD has joined.`

> ### - The PersistentVolumeClaim is optional, and is set to maintain the configuration of the deployed POD, so that in case the POD is destroyed and a new one is deployed, this new POD will connect to the ZeroTier network using the same configuration as before. That way it is not added as a new member within the network. The ZeroTier One service keeps its configuration and state information in its working directory. It’s found by default at the following location `/var/lib/zerotier-one`. This is where the Persistent Volume is mounted.
```console
root@zerotier-5cf9d447c-zhf14:/# ls -l /var/lib/zerotier-one/
total 36
-rw------- 1 zerotier-one zerotier-one   33 Feb  6 19:13 authtoken.secret
drwx------ 4 zerotier-one zerotier-one 4096 Feb  6 18:21 controller.d
-rw-r--r-- 1 zerotier-one zerotier-one  141 Feb  6 18:21 identity.public
-rw------- 1 zerotier-one zerotier-one  270 Feb  6 18:21 identity.secret
drwxr-xr-x 2 zerotier-one zerotier-one 4096 Feb  6 18:21 networks.d
drwxr-xr-x 2 zerotier-one zerotier-one 4096 Feb  6 20:27 peers.d
-rw-r--r-- 1 zerotier-one zerotier-one  570 Feb  6 18:21 planet
-rw-r--r-- 1 zerotier-one zerotier-one    2 Feb  6 19:16 zerotier-one.pid
-rw------- 1 zerotier-one zerotier-one    4 Feb  6 19:16 zerotier-one.port
```