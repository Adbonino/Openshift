# SELECTOR DE NODO

## Selector de nodo: valido para todo el cluster

```
[root@bastion NodoSelector]# oc project
Using project "demo" on server "https://api.prev.la.logicalis.com:6443".
[root@bastion NodoSelector]# oc apply -f deploy.yaml
deployment.apps/example created
[root@bastion NodoSelector]#
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
  replicas: 4
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



Veo donde estan programados los Pods.
```
[root@bastion NodoSelector]# oc get pods -o wide
NAME                      READY   STATUS    RESTARTS   AGE   IP             NODE                      NOMINATED NODE   READINESS GATES
example-7fdf6f7cb-8cph2   1/1     Running   0          92s   10.131.1.137   prev-rkh2g-worker-s6grv   <none>           <none>
example-7fdf6f7cb-fvkx6   1/1     Running   0          92s   10.131.1.136   prev-rkh2g-worker-s6grv   <none>           <none>
example-7fdf6f7cb-sm8jx   1/1     Running   0          92s   10.128.2.62    prev-rkh2g-worker-jfklf   <none>           <none>
example-7fdf6f7cb-tsnmt   1/1     Running   0          92s   10.128.2.61    prev-rkh2g-worker-jfklf   <none>           <none>
[root@bastion NodoSelector]#
```

Agrego una etiqueta a cada nodo:
```
[root@bastion NodoSelector]# oc get nodes
NAME                      STATUS   ROLES    AGE    VERSION
prev-rkh2g-master-0       Ready    master   313d   v1.20.11+aa025a0
prev-rkh2g-master-1       Ready    master   313d   v1.20.11+aa025a0
prev-rkh2g-master-2       Ready    master   313d   v1.20.11+aa025a0
prev-rkh2g-worker-jfklf   Ready    worker   313d   v1.20.11+aa025a0
prev-rkh2g-worker-s6grv   Ready    worker   313d   v1.20.11+aa025a0
[root@bastion NodoSelector]# oc label node prev-rkh2g-worker-jfklf Nodo=tipo1
node/prev-rkh2g-worker-jfklf labeled
[root@bastion NodoSelector]# oc label node prev-rkh2g-worker-s6grv Nodo=tipo2
node/prev-rkh2g-worker-s6grv labeled
[root@bastion NodoSelector]#
```
Veoq el nodo para ver la etiqueta:

```
[root@bastion NodoSelector]# oc get node  prev-rkh2g-worker-s6grv -o yaml | less
apiVersion: v1
kind: Node
metadata:
  annotations:
    machine.openshift.io/machine: openshift-machine-api/prev-rkh2g-worker-s6grv
    machineconfiguration.openshift.io/currentConfig: rendered-worker-85fb56456395777da9e9122c61aace71
    machineconfiguration.openshift.io/desiredConfig: rendered-worker-85fb56456395777da9e9122c61aace71
    machineconfiguration.openshift.io/reason: ""
    machineconfiguration.openshift.io/state: Done
    volumes.kubernetes.io/controller-managed-attach-detach: "true"
  creationTimestamp: "2021-05-13T20:04:01Z"
  labels:
    Nodo: tipo2
    beta.kubernetes.io/arch: amd64
    beta.kubernetes.io/os: linux
    kubernetes.io/arch: amd64
    kubernetes.io/hostname: prev-rkh2g-worker-s6grv
    kubernetes.io/os: linux
    node-role.kubernetes.io/worker: ""
    node.openshift.io/os_id: rhcos
    topology.hostpath.csi/node: prev-rkh2g-worker-s6grv
    topology.kubernetes.io/zone: west
[root@bastion NodoSelector]# oc get nodes -l Nodo=tipo1
NAME                      STATUS   ROLES    AGE    VERSION
prev-rkh2g-worker-jfklf   Ready    worker   313d   v1.20.11+aa025a0
[root@bastion NodoSelector]# oc get nodes -l Nodo=tipo2
NAME                      STATUS   ROLES    AGE    VERSION
prev-rkh2g-worker-s6grv   Ready    worker   313d   v1.20.11+aa025a0
[root@bastion NodoSelector]#
```

Puedo ver que hay dos Pods en cada nodo
```
[root@bastion NodoSelector]# oc get pods -o wide
NAME                      READY   STATUS    RESTARTS   AGE    IP             NODE                      NOMINATED NODE   READINESS GATES
example-7fdf6f7cb-8cph2   1/1     Running   0          8m2s   10.131.1.137   prev-rkh2g-worker-s6grv   <none>           <none>
example-7fdf6f7cb-fvkx6   1/1     Running   0          8m2s   10.131.1.136   prev-rkh2g-worker-s6grv   <none>           <none>
example-7fdf6f7cb-sm8jx   1/1     Running   0          8m2s   10.128.2.62    prev-rkh2g-worker-jfklf   <none>           <none>
example-7fdf6f7cb-tsnmt   1/1     Running   0          8m2s   10.128.2.61    prev-rkh2g-worker-jfklf   <none>           <none>
[root@bastion NodoSelector]#
```

Agrego el nodo selector al deployment:

- OPCION1: comando patch, 
- OPCOIN2: agregar las lines:
      nodeSelector:
        Nodo: tipo1
- OPCION3: edito el deployment y agregarle las dos mismas lineas que en el paso anterior
```
[root@bastion NodoSelector]# oc get all
NAME                          READY   STATUS    RESTARTS   AGE
pod/example-7fdf6f7cb-8cph2   1/1     Running   0          9m43s
pod/example-7fdf6f7cb-fvkx6   1/1     Running   0          9m43s
pod/example-7fdf6f7cb-sm8jx   1/1     Running   0          9m43s
pod/example-7fdf6f7cb-tsnmt   1/1     Running   0          9m43s

NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/example   4/4     4            4           9m43s

NAME                                DESIRED   CURRENT   READY   AGE
replicaset.apps/example-7fdf6f7cb   4         4         4       9m43s
[root@bastion NodoSelector]# oc patch deployment.apps/example -n demo --patch '{"spec":{"template":{"spec":{"nodeSelector":{"Nodo":"tipo1"}}}}}'

[root@bastion NodoSelector]# oc get pods -o wide
NAME                       READY   STATUS    RESTARTS   AGE   IP            NODE                      NOMINATED NODE   READINESS GATES
example-6854446bb6-7684p   1/1     Running   0          15s   10.128.2.69   prev-rkh2g-worker-jfklf   <none>           <none>
example-6854446bb6-9vxd4   1/1     Running   0          14s   10.128.2.70   prev-rkh2g-worker-jfklf   <none>           <none>
example-6854446bb6-ptnzd   1/1     Running   0          21s   10.128.2.67   prev-rkh2g-worker-jfklf   <none>           <none>
example-6854446bb6-qc5vd   1/1     Running   0          20s   10.128.2.68   prev-rkh2g-worker-jfklf   <none>           <none>
[root@bastion NodoSelector]#
```


```
[root@bastion NodoSelector]# oc get deployment.apps/example -o yaml | less
[root@bastion NodoSelector]#
........
    spec:
      containers:
      - image: openshift/hello-openshift
        imagePullPolicy: Always
        name: hello-openshift
        ports:
        - containerPort: 8080
          protocol: TCP
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      nodeSelector:
        Nodo: tipo1
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
........
```

Que pasa si el Pods no encuentra un nodo etiquetado?

cambiemos las etiquetas en el deployment:
```
      nodeSelector:
        Nodo: tiponull
```
```
[root@bastion NodoSelector]# oc edit deployment.apps/example
.............
    spec:
      containers:
      - image: openshift/hello-openshift
        imagePullPolicy: Always
        name: hello-openshift
        ports:
        - containerPort: 8080
          protocol: TCP
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      nodeSelector:
        Nodo: tiponull
...........
```
```
[root@bastion NodoSelector]# oc delete all --all
[root@bastion NodoSelector]# oc apply -f deploy.yaml
[root@bastion NodoSelector]# oc get pods
[root@bastion NodoSelector]# oc get pods
NAME                       READY   STATUS    RESTARTS   AGE
example-78548f5f78-2k8br   0/1     Pending   0          50s
example-78548f5f78-kzrd5   0/1     Pending   0          50s
example-78548f5f78-scwn2   0/1     Pending   0          50s
example-78548f5f78-sxqwz   0/1     Pending   0          50s
[root@bastion NodoSelector]#
[root@bastion NodoSelector]# oc get events | grep -i warning
70s         Warning   FailedScheduling    pod/example-78548f5f78-2k8br    0/5 nodes are available: 2 node(s) didn't match Pod's node affinity, 3 node(s) had taint {node-role.kubernetes.io/master: }, that the pod didn't tolerate.
70s         Warning   FailedScheduling    pod/example-78548f5f78-kzrd5    0/5 nodes are available: 2 node(s) didn't match Pod's node affinity, 3 node(s) had taint {node-role.kubernetes.io/master: }, that the pod didn't tolerate.
70s         Warning   FailedScheduling    pod/example-78548f5f78-scwn2    0/5 nodes are available: 2 node(s) didn't match Pod's node affinity, 3 node(s) had taint {node-role.kubernetes.io/master: }, that the pod didn't tolerate.
70s         Warning   FailedScheduling    pod/example-78548f5f78-sxqwz    0/5 nodes are available: 2 node(s) didn't match Pod's node affinity, 3 node(s) had taint {node-role.kubernetes.io/master: }, that the pod didn't tolerate.
[root@bastion NodoSelector]#
```
Podemos ver que hasta que le Pods no encuantre un nodo con la etiqueta adecuado el Scheduler no lo va a porgramar.


## Selector de nodos de proyectos: para colocar nuevos pods dentro de proyectos en nodos especificios

Agregamos el selector al proyecto.

```
[root@bastion NodoSelector]# oc annotate ns/demo openshift.io/node-selector="Nodo=tipo1"
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
  replicas: 4
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
[root@bastion NodoSelector]# oc apply -f deploy.yaml
deployment.apps/example created
```
Verificación
```
[root@bastion NodoSelector]# oc get pods -o wide
NAME                      READY   STATUS    RESTARTS   AGE   IP            NODE                      NOMINATED NODE   READINESS GATES
example-7fdf6f7cb-2dpjs   1/1     Running   0          14s   10.128.2.89   prev-rkh2g-worker-jfklf   <none>           <none>
example-7fdf6f7cb-7mrv5   1/1     Running   0          14s   10.128.2.88   prev-rkh2g-worker-jfklf   <none>           <none>
example-7fdf6f7cb-m7m6g   1/1     Running   0          14s   10.128.2.90   prev-rkh2g-worker-jfklf   <none>           <none>
example-7fdf6f7cb-n5twp   1/1     Running   0          14s   10.128.2.87   prev-rkh2g-worker-jfklf   <none>           <none>
[root@bastion NodoSelector]#
```

Borro la anotación.
```
[root@bastion NodoSelector]# oc annotate ns/demo openshift.io/node-selector-
```

## Etiquetas conocidas

Hay algunas etiquetas que se pueden aprovechar y algunas ya están predefinidas en los nodos:

    beta.kubernetes.io/arch=amd64

    kubernetes.io/hostname=compute-0

    kubernetes.io/os=linux

    node-role.kubernetes.io/worker=

    node.openshift.io/os_id=rhcos


Hay dos etiquetas que se pueden ustilizar para la ubicación de los Pods:

    topology.kubernetes.io/zone

    topology.kubernetes.io/region
