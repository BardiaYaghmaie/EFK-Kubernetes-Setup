# Kubernetes Logging with EFK (Elasticsearch, Fluentd & Kibana)

### by Bardia Yaghmaie

In this document we are going to setup EFK stack on our k8s cluster.

EFK is a popular open-source stack used for collecting, processing, and visualizing logs. The EFK stack is often used by developers and system administrators to gain insights into the behavior of their applications and infrastructure.

0- On control-plane with kubectl installed, create a directory for efk setup, including 4 directories for elasticsearch, fluentd, kibana and persistence.

```jsx
efk-setup/
		|_ elasticsearch/
		|					|_ statefulset.yaml
		|					|_ service.yaml
		|
		|_ kibana/
		|					|_ deployment.yaml
		|					|_ service.yaml
		|
		|_ fluentd/
		|					|_ clusterrole.yaml
		|					|_ serviceaccount.yaml
		|					|_ clusterrolebinding.yaml
		|					|_ daemonset.yaml
		|					
		|_ persistence/
							|_ storageclass.yaml
							|_ persistentvolume.yaml
```

1- cd into your persistence directory to apply storage class and persistent volume.

```jsx
# storageclass.yaml

kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: efk-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```

```jsx
kubectl apply -f storageclass.yaml
```

2- execute these commands on the worker node where the pv will be located.

```jsx
mkdir -p /mnt/data/efk
chmode 777 /mnt/data/efk
```

3- configure the persistent volume.

```jsx
# persistentvolume.yaml

apiVersion: v1
kind: PersistentVolume
metadata:
  name: efk-local-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: efk-storage
  local:
    path: /mnt/data/efk
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - <"hostname of the worker node">
```

```jsx
kubectl apply -f persistentvolume.yaml
```

4- cd into elasticsearch directory to apply statefulset and service.

```jsx
# statefulset.yaml

apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: es-cluster
spec:
  serviceName: elasticsearch
  replicas: 1
  selector:
    matchLabels:
      app: elasticsearch
  template:
    metadata:
      labels:
        app: elasticsearch
    spec:
      containers:
      - name: elasticsearch
        image: docker.elastic.co/elasticsearch/elasticsearch:7.14.0
        resources:
            limits:
              cpu: 1000m
            requests:
              cpu: 100m
        ports:
        - containerPort: 9200
          name: rest
          protocol: TCP
        - containerPort: 9300
          name: inter-node
          protocol: TCP
        volumeMounts:
        - name: data
          mountPath: /usr/share/elasticsearch/data
        env:
          - name: cluster.name
            value: k8s-logs
          - name: node.name
            valueFrom:
              fieldRef:
                fieldPath: metadata.name
          - name: discovery.seed_hosts
            value: "es-cluster-0.elasticsearch" 
            # for each master node add an element to this list (e.g 2 masters):
            # value: "es-cluster-0.elasticsearch,es-cluster-1.elasticsearch"
          - name: cluster.initial_master_nodes
            value: "es-cluster-0"
            # for each master node add an element to this list (e.g 2 masters):
            # value: "es-cluster-0, es-cluster-1"
          - name: ES_JAVA_OPTS
            value: "-Xms512m -Xmx512m"
      initContainers:
      - name: fix-permissions
        image: busybox
        command: ["sh", "-c", "chown -R 1000:1000 /usr/share/elasticsearch/data"]
        securityContext:
          privileged: true
        volumeMounts:
        - name: data
          mountPath: /usr/share/elasticsearch/data
      - name: increase-vm-max-map
        image: busybox
        command: ["sysctl", "-w", "vm.max_map_count=262144"]
        securityContext:
          privileged: true
      - name: increase-fd-ulimit
        image: busybox
        command: ["sh", "-c", "ulimit -n 65536"]
        securityContext:
          privileged: true
  volumeClaimTemplates:
  - metadata:
      name: data
      labels:
        app: elasticsearch
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "efk-storage"
      resources:
        requests:
          storage: 3Gi # <change this to meet your needs>
```

5- make sure if pvc and pv are bounded.

```jsx
$ kubectl get pvc

NAME                STATUS   VOLUME           CAPACITY   ACCESS MODES   STORAGECLASS     VOLUMEATTRIBUTESCLASS   AGE
data-es-cluster-0   Bound    efk-local-pv   10Gi       RWO            efk-storage      <unset>                 105m
```

```jsx
$ kubectl get pv 

NAME              CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM                            STORAGECLASS     VOLUMEATTRIBUTESCLASS   REASON   AGE
efk-local-pv      10Gi       RWO            Retain           Bound       default/data-es-cluster-0        efk-storage      <unset>                          110m
```

```jsx
$ kubectl get pod 

NAME                      READY   STATUS             RESTARTS        AGE
es-cluster-0              1/1     Running            0               167m
```

<aside>
💡 NOTE! 
These persistent volumes and claims can such a headache, if you don’t get the results as same as up here in the doc and see <PENDING> status, play around with pvc (delete and apply) to make it bound with the pv.

</aside>

6- apply the service for elasticsearch.

```jsx
# service.yaml

kind: Service
apiVersion: v1
metadata:
  name: elasticsearch
  labels:
    app: elasticsearch
spec:
  selector:
    app: elasticsearch
  clusterIP: None
  ports:
    - port: 9200
      name: rest
    - port: 9300
      name: inter-node
```

```jsx
kubectl apply -f service.yaml
```

7- check that the service is working fine.

```jsx
$ kubectl get svc

NAME            TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)             AGE
elasticsearch   ClusterIP   None          <none>        9200/TCP,9300/TCP   3h2m
```

```jsx
kubectl port-forward svc/elasticsearch 9200:9200 --address 127.0.0.1
```

```jsx
$ curl 127.0.0.1:9200

{
  "name" : "es-cluster-0",
  "cluster_name" : "k8s-logs",
  "cluster_uuid" : "******************",
  "version" : {
    "number" : "7.14.0",
    "build_flavor" : "default",
    "build_type" : "docker",
    "build_hash" : "************************8",
    "build_date" : "2021-07-29T20:49:32.864135063Z",
    "build_snapshot" : false,
    "lucene_version" : "8.9.0",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}

```

if you don’t see this output, review your installation.

8- cd into kibana directory to apply the deployment and service.

```jsx
# deployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: kibana
  labels:
    app: kibana
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kibana
  template:
    metadata:
      labels:
        app: kibana
    spec:
      containers:
      - name: kibana
        image: docker.elastic.co/kibana/kibana:7.14.0
        resources:
          limits:
            cpu: 1000m
          requests:
            cpu: 100m
        env:
          - name: ELASTICSEARCH_URL
            value: http://elasticsearch:9200
        ports:
        - containerPort: 5601
```

```jsx
kubectl apply -f deployment.yaml
```

```jsx
# service.yaml

apiVersion: v1
kind: Service
metadata:
  name: kibana
spec:
  selector: 
    app: kibana
  type: NodePort  
  ports:
    - port: 8080
      targetPort: 5601 
      nodePort: 30001

```

```jsx
kubectl apply -f service.yaml
```

9- cd into fluentd directory, here we set cluster roles, bindings, service account and the fluntd daemonset.

```jsx
#clusterrole.yaml

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: fluentd
  labels:
    app: fluentd
rules:
- apiGroups:
  - ""
  resources:
  - pods
  - namespaces
  verbs:
  - get
  - list
  - watch
```

```jsx
kubectl apply -f clusterrole.yaml
```

```jsx
	# serviceaccount.yaml
	
apiVersion: v1
kind: ServiceAccount
metadata:
  name: fluentd
  labels:
    app: fluentd
```

```jsx
kubectl apply -f serviceaccount.yaml
```

```jsx
# clusterrolebinding.yaml

kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: fluentd
roleRef:
  kind: ClusterRole
  name: fluentd
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: ServiceAccount
  name: fluentd
  namespace: default

```

```jsx
kubectl apply -f clusterrolebinding.yaml
```

```jsx
# daemonset.yaml

apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
  labels:
    app: fluentd
spec:
  selector:
    matchLabels:
      app: fluentd
  template:
    metadata:
      labels:
        app: fluentd
    spec:
      serviceAccount: fluentd
      serviceAccountName: fluentd
      containers:
      - name: fluentd
        image: fluent/fluentd-kubernetes-daemonset:v1.4.2-debian-elasticsearch-1.1
        env:
          - name:  FLUENT_ELASTICSEARCH_HOST
            value: "elasticsearch.default.svc.cluster.local"
          - name:  FLUENT_ELASTICSEARCH_PORT
            value: "9200"
          - name: FLUENT_ELASTICSEARCH_SCHEME
            value: "http"
          - name: FLUENTD_SYSTEMD_CONF
            value: disable
          - name: FLUENT_CONTAINER_TAIL_PARSER_TYPE
            value: /^(?<time>.+) (?<stream>stdout|stderr) [^ ]* (?<log>.*)$/
          - name: FLUENT_CONTAINER_TAIL_EXCLUDE_PATH
            value: /var/log/containers/fluent*
        resources:
          limits:
            memory: 512Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```

```jsx
kubectl apply -f daemonset.yaml
```

10- now it’s time to check everything is running.

```jsx
$ kubectl get all

NAME                          READY   STATUS             RESTARTS        AGE
pod/es-cluster-0              1/1     Running            0               3h28m
pod/fluentd-zcz9m             1/1     Running            0               3h25m
pod/kibana-5xxdd54f7c-xfq2l   1/1     Running            0               3h26m

NAME                    TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)             AGE
service/elasticsearch   ClusterIP   None          <none>        9200/TCP,9300/TCP   3h32m
service/kibana          NodePort    10.233.x.x   <none>        8080:30001/TCP      3h26m
service/kubernetes      ClusterIP   10.233.0.1    <none>        443/TCP             24h

NAME                     DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
daemonset.apps/fluentd   1         1         1       1            1           <none>          3h25m

NAME                     READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/kibana   1/1     1            1           3h26m

NAME                                DESIRED   CURRENT   READY   AGE
replicaset.apps/kibana-5xxdd54f7c   1         1         1       3h26m

NAME                          READY   AGE
statefulset.apps/es-cluster   1/1     3h28m
```

<aside>
☝🏽  **let’s test everything once**

</aside>

11- run an nginx pod.

```jsx
kubectl run nginx-efk-test --image=nginx --restart=Never
```

12- access kibana UI via its service <node-ip>:<node-port>

13- go to menu → management → stack management → index management

you should be able to see something like:
![Screenshot from 2024-03-17 14-46-15](https://github.com/BardiaYaghmaie/EFK-Kubernetes-Setup/assets/101035374/bbfb1b4b-424f-4575-b0c1-2edbc2e46d20)


14- go to menu → analytics → discover

in the search bar, search for nginx-efk-test and you should be able to see some logs
![Screenshot from 2024-03-17 14-51-10](https://github.com/BardiaYaghmaie/EFK-Kubernetes-Setup/assets/101035374/10eb241c-68e2-42e6-a087-8982f781a95c)


## Congrats! EFK is up and running!
