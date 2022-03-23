# USO DE MANCHAS Y TOLERANCIAS (taints / tolerations)


## LABORATORIO
```
[root@bastion ~]# oc get nodes
NAME                      STATUS   ROLES    AGE    VERSION
prev-rkh2g-master-0       Ready    master   313d   v1.20.11+aa025a0
prev-rkh2g-master-1       Ready    master   313d   v1.20.11+aa025a0
prev-rkh2g-master-2       Ready    master   313d   v1.20.11+aa025a0
prev-rkh2g-worker-jfklf   Ready    worker   313d   v1.20.11+aa025a0
prev-rkh2g-worker-s6grv   Ready    worker   313d   v1.20.11+aa025a0
[root@bastion ~]#
```

```
[root@bastion ~]# oc adm taint node prev-rkh2g-worker-jfklf key1=value1:NoSchedule
node/prev-rkh2g-worker-jfklf tainted
[root@bastion ~]#

[root@bastion ~]# oc get node prev-rkh2g-worker-jfklf -o yaml | less
spec:
  providerID: vsphere://420639c4-e675-8076-dd48-0e0a5bdfb30d
  // esto es lo que agrega el comando anterior.
  taints:
  - effect: NoSchedule
    key: key1
    value: value1
```

```
[root@bastion ~]# oc project zland
Already on project "zland" on server "https://api.prev.la.logicalis.com:6443".
[root@bastion ~]#
```
```
[root@bastion NodoSelector]# cat deploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: example
spec:
  selector:
    matchLabels:
      app: ejemplo-deploy
  replicas: 2
  template:
    metadata:
      labels:
        app: ejemplo-deploy
    spec:
        containers:
        - name: hello-openshift
          image: openshift/hello-openshift
          ports:
            - containerPort: 8080
```


```
[root@bastion NodoSelector]# oc apply -f deploy.yaml
deployment.apps/example created
```
// podemos ver que se programaron en el otro nodo
```
[root@bastion NodoSelector]# oc get all
NAME                          READY   STATUS    RESTARTS   AGE
pod/example-7fdf6f7cb-lqcx2   1/1     Running   0          15s
pod/example-7fdf6f7cb-tspww   1/1     Running   0          15s

NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/example   2/2     2            2           15s

NAME                                DESIRED   CURRENT   READY   AGE
replicaset.apps/example-7fdf6f7cb   2         2         2       15s
[root@bastion NodoSelector]# oc get pods -o wide
NAME                      READY   STATUS    RESTARTS   AGE   IP             NODE                      NOMINATED NODE   READINESS GATES
example-7fdf6f7cb-lqcx2   1/1     Running   0          32s   10.131.1.122   prev-rkh2g-worker-s6grv   <none>           <none>
example-7fdf6f7cb-tspww   1/1     Running   0          32s   10.131.1.123   prev-rkh2g-worker-s6grv   <none>           <none>
[root@bastion NodoSelector]#
```

//Mancho el segundo nodo
```
[root@bastion NodoSelector]# oc adm taint node prev-rkh2g-worker-s6grv key1=value1:NoSchedule
node/prev-rkh2g-worker-s6grv tainted
[root@bastion NodoSelector]#
```

//borro los pods
```
[root@bastion NodoSelector]# oc get pods
NAME                      READY   STATUS    RESTARTS   AGE
example-7fdf6f7cb-lqcx2   1/1     Running   0          2m11s
example-7fdf6f7cb-tspww   1/1     Running   0          2m11s
[root@bastion NodoSelector]# oc delete pods example-7fdf6f7cb-lqcx2 example-7fdf6f7cb-tspww
pod "example-7fdf6f7cb-lqcx2" deleted
pod "example-7fdf6f7cb-tspww" deleted
```
// veo que quedan en STATUS:Pending
```
[root@bastion NodoSelector]# oc get pods
NAME                      READY   STATUS    RESTARTS   AGE
example-7fdf6f7cb-fhxx9   0/1     Pending   0          17s
example-7fdf6f7cb-q42qc   0/1     Pending   0          17s
[root@bastion NodoSelector]# oc get pods -o wide
NAME                      READY   STATUS    RESTARTS   AGE   IP       NODE     NOMINATED NODE   READINESS GATES
example-7fdf6f7cb-fhxx9   0/1     Pending   0          23s   <none>   <none>   <none>           <none>
example-7fdf6f7cb-q42qc   0/1     Pending   0          23s   <none>   <none>   <none>           <none>
[root@bastion NodoSelector]#

[root@bastion NodoSelector]# oc get events | grep -i warning
......
3m15s       Warning   FailedScheduling    pod/example-7fdf6f7cb-fhxx9     0/5 nodes are available: 2 node(s) had taint {key1: value1}, that the pod didn't tolerate, 3 node(s) had taint {node-role.kubernetes.io/master: }, that the pod didn't tolerate.
......
3m15s       Warning   FailedScheduling    pod/example-7fdf6f7cb-q42qc     0/5 nodes are available: 2 node(s) had taint {key1: value1}, that the pod didn't tolerate, 3 node(s) had taint {node-role.kubernetes.io/master: }, that the pod didn't tolerate.
......
[root@bastion NodoSelector]#
```

Agrego la tolerancia al deployment
Modifico el manifiesto deploy.yaml y lo vuelvo a aplicar o reemplazar (Opcion1),o edito el deploymente actual (OPTION 2).
Entiendo que es mejor modificar el deploy actual.

OPCION 1
Agrego en el deploy.yaml las tolerations.

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: example
spec:
  selector:
    matchLabels:
      app: ejemplo-deploy
  replicas: 2
  template:
    metadata:
      labels:
        app: ejemplo-deploy
    spec:
        tolerations:
          - key: "key1"
            value: "value1"
            operator: "Equal"
            effect: "NoSchedule"
        containers:
        - name: hello-openshift
          image: openshift/hello-openshift
          ports:
            - containerPort: 8080
```
```
[root@bastion NodoSelector]# oc replace -f deploy.yaml
deployment.apps/example replaced
[root@bastion NodoSelector]#
[root@bastion NodoSelector]# oc get all
NAME                           READY   STATUS    RESTARTS   AGE
pod/example-7b79d6bf87-j8cm5   1/1     Running   0          10s
pod/example-7b79d6bf87-n4khn   1/1     Running   0          16s

NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/example   2/2     2            2           11m

NAME                                 DESIRED   CURRENT   READY   AGE
replicaset.apps/example-7b79d6bf87   2         2         2       16s
replicaset.apps/example-7fdf6f7cb    0         0         0       11m
[root@bastion NodoSelector]#
```

OPCION 2
Edito el deploy actual y le agrego las tolerancias.

```
[root@bastion NodoSelector]# oc get all
NAME                          READY   STATUS    RESTARTS   AGE
pod/example-7fdf6f7cb-rv9xd   0/1     Pending   0          4s
pod/example-7fdf6f7cb-tc44r   0/1     Pending   0          4s

NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/example   0/2     2            0           4s

NAME                                DESIRED   CURRENT   READY   AGE
replicaset.apps/example-7fdf6f7cb   2         2         0       4s
[root@bastion NodoSelector]#
[root@bastion NodoSelector]# oc edit deployment.apps/example
.............
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: ejemplo-deploy
    spec:
      tolerations:
          - key: "key1"
            value: "value1"
            operator: "Equal"
            effect: "NoSchedule"
      containers:
      - image: openshift/hello-openshift
        imagePullPolicy: Always
        name: hello-openshift
        ports:
.............
```

Verifico que se vuelvan a deplegar los pods.

```
[root@bastion NodoSelector]# oc get all
NAME                           READY   STATUS              RESTARTS   AGE
pod/example-7b79d6bf87-fv5f5   0/1     ContainerCreating   0          4s
pod/example-7fdf6f7cb-rv9xd    0/1     Pending             0          12m
pod/example-7fdf6f7cb-tc44r    0/1     Pending             0          12m

NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/example   0/2     1            0           12m

NAME                                 DESIRED   CURRENT   READY   AGE
replicaset.apps/example-7b79d6bf87   1         1         0       4s
replicaset.apps/example-7fdf6f7cb    2         2         0       12m
[root@bastion NodoSelector]#
```

```
[root@bastion NodoSelector]# oc get all
NAME                           READY   STATUS    RESTARTS   AGE
pod/example-7b79d6bf87-dd82w   1/1     Running   0          17s
pod/example-7b79d6bf87-fv5f5   1/1     Running   0          23s

NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/example   2/2     2            2           13m

NAME                                 DESIRED   CURRENT   READY   AGE
replicaset.apps/example-7b79d6bf87   2         2         2       23s
replicaset.apps/example-7fdf6f7cb    0         0         0       13m
[root@bastion NodoSelector]#
```

Borro las manchas en los nodos para dejarlos con su configuración inicial
```
[root@bastion NodoSelector]# oc adm taint node prev-rkh2g-worker-s6grv key1=value1:NoSchedule-
node/prev-rkh2g-worker-s6grv untainted
[root@bastion NodoSelector]# oc adm taint node prev-rkh2g-worker-jfklf key1=value1:NoSchedule-
node/prev-rkh2g-worker-jfklf untainted
[root@bastion NodoSelector]#
```
