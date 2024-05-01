專案地址：
https://github.com/StarRocks/starrocks
https://github.com/StarRocks/starrocks-kubernetes-operator

![](media/Pasted-image-20240429213348.png)

### Install the MinIO Operator using Local Helm Charts

參考Youtube影片：
https://www.youtube.com/watch?v=26CQyBkghrU

參考Minio安裝文件：
https://min.io/docs/minio/kubernetes/upstream/operations/install-deploy-manage/deploy-operator-helm.html#install-the-minio-operator-using-local-helm-charts

#### 1. Download the Helm charts

```bash
root@k8s-master71u:~# curl -O https://raw.githubusercontent.com/minio/operator/master/helm-releases/operator-5.0.14.tgz
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 21254  100 21254    0     0  32069      0 --:--:-- --:--:-- --:--:-- 32105
```

#### 2. Deploy Operator

```bash
root@k8s-master71u:~# tar -xzvf operator-5.0.14.tgz
```

修改values
```
修改以下三個地方

replicaCount: 3
env
ingress
```

```bash
root@k8s-master71u:~# vim operator/values.yaml

operator:
  env:

    - name: MINIO_OPERATOR_RUNTIME
      value: "Rancher"

  replicaCount: 3

  ingress:
    enabled: true
    ingressClassName: "nginx"
    labels: { }
    annotations: { }
    tls: [ ]
    host: minio.jimmyhome.tw
    path: /
    pathType: Prefix
```

```bash
root@k8s-master71u:~# helm install \
--namespace minio-operator \
--create-namespace \
minio-operator operator-5.0.14.tgz --values operator/values.yaml
NAME: minio-operator
LAST DEPLOYED: Wed May  1 11:49:45 2024
NAMESPACE: minio-operator
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
1. Get the JWT for logging in to the console:
kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: console-sa-secret
  namespace: minio-operator
  annotations:
    kubernetes.io/service-account.name: console-sa
type: kubernetes.io/service-account-token
EOF
kubectl -n minio-operator get secret console-sa-secret -o jsonpath="{.data.token}" | base64 --decode

2. Get the Operator Console URL by running these commands:
  kubectl --namespace minio-operator port-forward svc/console 9090:9090
  echo "Visit the Operator Console at http://127.0.0.1:9090"
```

```bash
root@k8s-master71u:~# helm list -n minio-operator
NAME          	NAMESPACE     	REVISION	UPDATED                                	STATUS  	CHART          	APP VERSION
minio-operator	minio-operator	1       	2024-05-01 11:49:45.278900358 +0000 UTC	deployed	operator-5.0.14	v5.0.14
```

```bash
root@k8s-master71u:~# kubectl get all -n minio-operator
NAME                                  READY   STATUS    RESTARTS   AGE
pod/console-dd6bbbd65-fbf9g           1/1     Running   0          2m36s
pod/minio-operator-5bf8757469-nvmff   1/1     Running   0          2m36s
pod/minio-operator-5bf8757469-pj42h   1/1     Running   0          2m36s
pod/minio-operator-5bf8757469-z9t4z   1/1     Running   0          2m36s

NAME               TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)             AGE
service/console    ClusterIP   10.202.113.242   <none>        9090/TCP,9443/TCP   2m36s
service/operator   ClusterIP   10.202.210.120   <none>        4221/TCP            2m36s
service/sts        ClusterIP   10.202.93.160    <none>        4223/TCP            2m36s

NAME                             READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/console          1/1     1            1           2m36s
deployment.apps/minio-operator   3/3     3            3           2m36s

NAME                                        DESIRED   CURRENT   READY   AGE
replicaset.apps/console-dd6bbbd65           1         1         1       2m36s
replicaset.apps/minio-operator-5bf8757469   3         3         3       2m36s
```
#### 3. Configure Operator

##### service.yaml
```bash
root@k8s-master71u:~# kubectl get service console -n minio-operator -o yaml > operator/service.yaml
```

新增nodePort: 30080、type: NodePort，以利後續使用nodePort訪問minio介面UI

```bash
root@k8s-master71u:~# cat operator/service.yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    meta.helm.sh/release-name: minio-operator
    meta.helm.sh/release-namespace: minio-operator
  creationTimestamp: "2024-05-01T11:49:49Z"
  labels:
    app.kubernetes.io/instance: minio-operator
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: operator
    app.kubernetes.io/version: v5.0.14
    helm.sh/chart: operator-5.0.14
  name: console
  namespace: minio-operator
  resourceVersion: "16451110"
  uid: 610edc4a-c6d0-4cb8-8631-2db6f28125a1
spec:
  internalTrafficPolicy: Cluster
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  ports:
  - name: http
    port: 9090
    protocol: TCP
    targetPort: 9090
    nodePort: 30080
  - name: https
    port: 9443
    protocol: TCP
    targetPort: 9443
  selector:
    app.kubernetes.io/instance: minio-operator-console
    app.kubernetes.io/name: operator
  sessionAffinity: None
  type: NodePort
status:
  loadBalancer: {}
```

```bash
root@k8s-master71u:~# kubectl apply -f operator/service.yaml
Warning: resource services/console is missing the kubectl.kubernetes.io/last-applied-configuration annotation which is required by kubectl apply. kubectl apply should only be used on resources created declaratively by either kubectl create --save-config or kubectl apply. The missing annotation will be patched automatically.
service/console configuredd
```

```bash
root@k8s-master71u:~# kubectl get all -n minio-operator
NAME                                  READY   STATUS    RESTARTS   AGE
pod/console-dd6bbbd65-fbf9g           1/1     Running   0          6m19s
pod/minio-operator-5bf8757469-nvmff   1/1     Running   0          6m19s
pod/minio-operator-5bf8757469-pj42h   1/1     Running   0          6m19s
pod/minio-operator-5bf8757469-z9t4z   1/1     Running   0          6m19s

NAME               TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                         AGE
service/console    NodePort    10.202.113.242   <none>        9090:30080/TCP,9443:32180/TCP   6m19s
service/operator   ClusterIP   10.202.210.120   <none>        4221/TCP                        6m19s
service/sts        ClusterIP   10.202.93.160    <none>        4223/TCP                        6m19s

NAME                             READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/console          1/1     1            1           6m19s
deployment.apps/minio-operator   3/3     3            3           6m19s

NAME                                        DESIRED   CURRENT   READY   AGE
replicaset.apps/console-dd6bbbd65           1         1         1       6m19s
replicaset.apps/minio-operator-5bf8757469   3         3         3       6m19s
```

##### operator.yaml


```bash
root@k8s-master71u:~# kubectl get deployment minio-operator -n minio-operator -o yaml > operator/operator.yaml
```

可以去修改replicas，預設為兩個，我一開始部署時，已經修改values，讓他生成3個

```bash
root@k8s-master71u:~# cat operator/operator.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "1"
    meta.helm.sh/release-name: minio-operator
    meta.helm.sh/release-namespace: minio-operator
  creationTimestamp: "2024-05-01T11:49:49Z"
  generation: 1
  labels:
    app.kubernetes.io/instance: minio-operator
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: operator
    app.kubernetes.io/version: v5.0.14
    helm.sh/chart: operator-5.0.14
  name: minio-operator
  namespace: minio-operator
  resourceVersion: "16451228"
  uid: 0888ecde-172e-4650-8210-ab94361dc2e9
spec:
  progressDeadlineSeconds: 600
  replicas: 3
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app.kubernetes.io/instance: minio-operator
      app.kubernetes.io/name: operator
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        app.kubernetes.io/instance: minio-operator
        app.kubernetes.io/name: operator
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: name
                operator: In
                values:
                - minio-operator
            topologyKey: kubernetes.io/hostname
      containers:
      - args:
        - controller
        env:
        - name: OPERATOR_STS_ENABLED
          value: "on"
        image: quay.io/minio/operator:v5.0.14
        imagePullPolicy: IfNotPresent
        name: operator
        resources:
          requests:
            cpu: 200m
            ephemeral-storage: 500Mi
            memory: 256Mi
        securityContext:
          runAsGroup: 1000
          runAsNonRoot: true
          runAsUser: 1000
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext:
        fsGroup: 1000
        runAsGroup: 1000
        runAsNonRoot: true
        runAsUser: 1000
      serviceAccount: minio-operator
      serviceAccountName: minio-operator
      terminationGracePeriodSeconds: 30
status:
  availableReplicas: 3
  conditions:
  - lastTransitionTime: "2024-05-01T11:49:52Z"
    lastUpdateTime: "2024-05-01T11:49:52Z"
    message: Deployment has minimum availability.
    reason: MinimumReplicasAvailable
    status: "True"
    type: Available
  - lastTransitionTime: "2024-05-01T11:49:49Z"
    lastUpdateTime: "2024-05-01T11:49:52Z"
    message: ReplicaSet "minio-operator-5bf8757469" has successfully progressed.
    reason: NewReplicaSetAvailable
    status: "True"
    type: Progressing
  observedGeneration: 1
  readyReplicas: 3
  replicas: 3
  updatedReplicas: 3
```

如果operator.yaml沒異動，就不用apply了

```bash
root@k8s-master71u:~/minio# kubectl apply -f operator.yaml
```

##### console-secret.yaml

創建secret，產生token，GUI登入頁面使用，
一開始，helm install完成時，有輸出以下內容，叫我們建立console-sa-secret，
就是我現在要執行的步驟

```bash
root@k8s-master71u:~# vim operator/console-sa-secret.yaml

apiVersion: v1
kind: Secret
metadata:
  name: console-sa-secret
  namespace: minio-operator
  annotations:
    kubernetes.io/service-account.name: console-sa
type: kubernetes.io/service-account-token
```

```bash
root@k8s-master71u:~# kubectl apply -f operator/console-sa-secret.yaml
secret/console-sa-secret created
```

```bash
root@k8s-master71u:~# kubectl get secrets -n minio-operator
NAME                                   TYPE                                  DATA   AGE
console-sa-secret                      kubernetes.io/service-account-token   3      2s
sh.helm.release.v1.minio-operator.v1   helm.sh/release.v1                    1      13m
sts-tls                                Opaque                                2      12m
```
#### 4. Connect to the Operator Console

A. Retrieve the JWT

```bash
root@k8s-master71u:~# SA_TOKEN=$(kubectl -n minio-operator  get secret console-sa-secret -o jsonpath="{.data.token}" | base64 --decode)

root@k8s-master71u:~# echo $SA_TOKEN
eyJhbGciOiJSUzI1NiIsImtpZCI6IjYyUGlOU1JjdU96N1hPRzN1X1FkYjVJZGJzRW9obENaWjhka015TV9TMmcifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJtaW5pby1vcGVyYXRvciIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJjb25zb2xlLXNhLXNlY3JldCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50Lm5hbWUiOiJjb25zb2xlLXNhIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiYzg5OTA0OWYtYmI4ZS00NzE1LTk0MDgtZWJlNWJmY2Q3Nzk3Iiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50Om1pbmlvLW9wZXJhdG9yOmNvbnNvbGUtc2EifQ.otAIGmGju8nD1NDg5FjrOiNubSbXtmsD4mOGJl3MQoNWiTAY91Xyj_a1lpLt8wrImi8BEFvp3o6_kBmLU71yeZTuveFO7uVaY4zZ2H2WtsWDEY9eYywDEsd0vUTcFIwWsGt29fWEeEtsBls6oPDEB4ld7C0YzdKptKPrRtrKLzhYd_MoaDoO5D1NdelUuRPxTbVoLfOGoXTbOvreA0owVJmTtjgoz_SzM_VA-Ix0uEBODC8KJ8ZjjCE9rHkWfHyYzXv4_ZydGghaZMCAX2wwZ4Ez_6KJLhNnmekYWyx9_TA3U9Bbg2vkmL-VmTTFnU1BHyjI-X1UIsjFUWV74Icz9Q
```


```bash
root@k8s-master71u:~# kubectl get all -n minio-operator
NAME                                  READY   STATUS    RESTARTS   AGE
pod/console-dd6bbbd65-fbf9g           1/1     Running   0          14m
pod/minio-operator-5bf8757469-nvmff   1/1     Running   0          14m
pod/minio-operator-5bf8757469-pj42h   1/1     Running   0          14m
pod/minio-operator-5bf8757469-z9t4z   1/1     Running   0          14m

NAME               TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                         AGE
service/console    NodePort    10.202.113.242   <none>        9090:30080/TCP,9443:32180/TCP   14m
service/operator   ClusterIP   10.202.210.120   <none>        4221/TCP                        14m
service/sts        ClusterIP   10.202.93.160    <none>        4223/TCP                        14m

NAME                             READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/console          1/1     1            1           14m
deployment.apps/minio-operator   3/3     3            3           14m

NAME                                        DESIRED   CURRENT   READY   AGE
replicaset.apps/console-dd6bbbd65           1         1         1       14m
replicaset.apps/minio-operator-5bf8757469   3         3         3       14m
```

```bash
root@k8s-master71u:~# kubectl get ingress -n minio-operator
NAME                     CLASS   HOSTS                ADDRESS   PORTS   AGE
minio-operator-console   nginx   minio.jimmyhome.tw             80      14m
```

本機設定hosts解析，測試ingress
```bash
chenqingze@chenqingze-MBP ~ % sudo vim /etc/hosts

192.168.1.75 minio.jimmyhome.tw
```

域名解析minio.jimmyhome.tw

![](media/Pasted-image-20240501200707.png)

使用Service NodePort訪問

![](media/Pasted-image-20240501200810.png)

token登入
![](media/Pasted-image-20240501200911.png)
![](media/Pasted-image-20240501200953.png)

#### 5. Creating Kubernetes Tenants Using Helm Lab Walkthrough

參考Youtube影片：
https://www.youtube.com/watch?v=yjQxYw8ZBGU

參考Tenants部署文件：
https://min.io/docs/minio/kubernetes/upstream/operations/deploy-manage-tenants.html

```bash
root@k8s-master71u:~/operator# curl -O https://raw.githubusercontent.com/minio/operator/master/helm-releases/tenant-5.0.14.tgz
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 10162  100 10162    0     0  32629      0 --:--:-- --:--:-- --:--:-- 32675
```

```bash
root@k8s-master71u:~/operator# tar -xzvf tenant-5.0.14.tgz

root@k8s-master71u:~/operator# cd tenant/

root@k8s-master71u:~/operator/tenant# ls -l
total 32
-rw-r--r-- 1 root root   348 Mar 15 19:18 Chart.yaml
-rw-r--r-- 1 root root  1169 Mar 15 19:18 README.md
drwxr-xr-x 2 root root  4096 May  1 12:27 templates
-rw-r--r-- 1 root root 20043 Mar 15 19:18 values.yaml
```

```yaml
root@k8s-master71u:~/operator/tenant# vim values.yaml


secrets:
  name: tenant1-env-configuration
  accessKey: minio
  secretKey: minio123

tenant:
  ###
  # The Tenant name
  #
  # Change this to match your preferred MinIO Tenant name.
  name: tenant1

  configuration:
    name: tenant1-env-configuration

  pools:
    ###
    # The number of MinIO Tenant Pods / Servers in this pool.
    # For standalone mode, supply 1. For distributed mode, supply 4 or more.
    # Note that the operator does not support upgrading from standalone to distributed mode.
    - servers: 1
      ###
      # Custom name for the pool
      name: pool-0
      ###
      # The number of volumes attached per MinIO Tenant Pod / Server.
      volumesPerServer: 1
      ###
      # The capacity per volume requested per MinIO Tenant Pod.
      size: 50Gi
      ###
      # The `storageClass <https://kubernetes.io/docs/concepts/storage/storage-classes/>`__ to associate with volumes generated for this pool.
      #
      # If using Amazon Elastic Block Store (EBS) CSI driver
      # Please make sure to set xfs for "csi.storage.k8s.io/fstype" parameter under StorageClass.parameters.
      # Docs: https://github.com/kubernetes-sigs/aws-ebs-csi-driver/blob/master/docs/parameters.md
      storageClassName: rook-cephfs
```


```bash
root@k8s-master71u:~/operator/tenant# kubectl get pod -n minio-operator
NAME                              READY   STATUS    RESTARTS   AGE
console-dd6bbbd65-fbf9g           1/1     Running   0          45m
minio-operator-5bf8757469-nvmff   1/1     Running   0          45m
minio-operator-5bf8757469-pj42h   1/1     Running   0          45m
minio-operator-5bf8757469-z9t4z   1/1     Running   0          45m
```


```bash
root@k8s-master71u:~/operator/tenant# cd ..
root@k8s-master71u:~/operator# helm install \
> --namespace tenant1 \
> --create-namespace \
> tenant1 tenant-5.0.14.tgz --values tenant/values.yaml
NAME: tenant1
LAST DEPLOYED: Wed May  1 13:14:56 2024
NAMESPACE: tenant1
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
To connect to the tenant1 tenant if it doesn't have a service exposed, you can port-forward to it by running:

  kubectl --namespace tenant1 port-forward svc/tenant1-console 9443:9443

  Then visit the MinIO Console at https://127.0.0.1:9443
```


```bash
root@k8s-master71u:~/operator# kubectl get all -n tenant1
NAME                   READY   STATUS    RESTARTS   AGE
pod/tenant1-pool-0-0   2/2     Running   0          28s

NAME                      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/minio             ClusterIP   10.202.14.112   <none>        443/TCP    29s
service/tenant1-console   ClusterIP   10.202.6.203    <none>        9443/TCP   29s
service/tenant1-hl        ClusterIP   None            <none>        9000/TCP   28s

NAME                              READY   AGE
statefulset.apps/tenant1-pool-0   1/1     28s
```


```bash
root@k8s-master71u:~/operator# kubectl get service tenant1-console -n tenant1 -o yaml > tenant/tenant1-service.yaml
root@k8s-master71u:~/operator# vim tenant/tenant1-service.yaml

apiVersion: v1
kind: Service
metadata:
  creationTimestamp: "2024-05-01T13:15:12Z"
  name: tenant1-console
  namespace: tenant1
  ownerReferences:
  - apiVersion: minio.min.io/v2
    blockOwnerDeletion: true
    controller: true
    kind: Tenant
    name: tenant1
    uid: d8a7eec8-7c09-4445-9f36-8688bcc218a9
  resourceVersion: "16499011"
  uid: a977f387-c0ef-4d91-bc3b-eefa36cceb75
spec:
  internalTrafficPolicy: Cluster
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  ports:
  - name: https-console
    port: 9443
    protocol: TCP
    targetPort: 9443
    nodePort: 30443
  selector:
    v1.min.io/tenant: tenant1
  sessionAffinity: None
  type: NodePort
status:
  loadBalancer: {}
```

```bash
root@k8s-master71u:~/operator# kubectl apply -f tenant/tenant1-service.yaml
Warning: resource services/tenant1-console is missing the kubectl.kubernetes.io/last-applied-configuration annotation which is required by kubectl apply. kubectl apply should only be used on resources created declaratively by either kubectl create --save-config or kubectl apply. The missing annotation will be patched automatically.
service/tenant1-console configured
```

```bash
root@k8s-master71u:~/operator# kubectl get all -n tenant1
NAME                   READY   STATUS    RESTARTS   AGE
pod/tenant1-pool-0-0   2/2     Running   0          8m18s

NAME                      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
service/minio             ClusterIP   10.202.14.112   <none>        443/TCP          8m19s
service/tenant1-console   NodePort    10.202.6.203    <none>        9443:30443/TCP   8m19s
service/tenant1-hl        ClusterIP   None            <none>        9000/TCP         8m18s

NAME                              READY   AGE
statefulset.apps/tenant1-pool-0   1/1     8m18s
```

https://192.168.1.75:30443/

![](media/Pasted-image-20240501212456.png)

![](media/Pasted-image-20240501212525.png)

![](media/Pasted-image-20240501212609.png)

![](media/Pasted-image-20240501212926.png)

回到minio-operator，可以看到有tenant1了
http://minio.jimmyhome.tw/tenants

![](media/Pasted-image-20240501212849.png)

創建buckets，並上傳資料測試，同時下載測試

![](media/Pasted-image-20240501213212.png)

上傳
![](media/Pasted-image-20240501213327.png)

上傳成功
![](media/Pasted-image-20240501213356.png)

下載成功
![](media/Pasted-image-20240501213506.png)


### 部署 StarRocks Operator

安裝SOP：
https://docs.starrocks.io/zh/docs/deployment/sr_operator/

k8s使用ceph作為動態storageClass儲存

1. 新增自訂資源StarRocksCluster。

```bash
root@k8s-master71u:~# kubectl apply -f https://raw.githubusercontent.com/StarRocks/starrocks-kubernetes-operator/main/deploy/starrocks.com_starrocksclusters.yaml
customresourcedefinition.apiextensions.k8s.io/starrocksclusters.starrocks.com created
```

2. 部署StarRocks Operator。您可以選擇使用預設設定檔或自訂設定檔部署StarRocks Operator。

i. 使用預設設定檔部署StarRocks Operator：
```bash
root@k8s-master71u:~# kubectl apply -f https://raw.githubusercontent.com/StarRocks/starrocks-kubernetes-operator/main/deploy/operator.yaml
namespace/starrocks created
serviceaccount/starrocks created
clusterrole.rbac.authorization.k8s.io/kube-starrocks-operator created
clusterrolebinding.rbac.authorization.k8s.io/kube-starrocks-operator created
role.rbac.authorization.k8s.io/cn-leader-election-role created
rolebinding.rbac.authorization.k8s.io/cn-leader-election-rolebinding created
deployment.apps/kube-starrocks-operator created
```

3. 檢查StarRocks Operator 的運作狀態。如果Pod 處於`Running`狀態且Pod 內所有容器都`READY`，則表示StarRocks Operator 成功運作。

```bash
root@k8s-master71u:~# kubectl -n starrocks get pods
NAME                                       READY   STATUS    RESTARTS   AGE
kube-starrocks-operator-574b859d5f-qjxcl   1/1     Running   0          51s
```

### 部署 StarRocks 集群

存算分離FE+CN+storageClass做FE元數據持久化


```yaml
root@k8s-master71u:~/starrocks# cat shared_data_mode.yaml
apiVersion: starrocks.com/v1
kind: StarRocksCluster
metadata:
  name: starrockscluster
  namespace: starrocks
spec:
  starRocksFeSpec:
    image: starrocks/fe-ubuntu:latest
    replicas: 3
    limits:
      cpu: 2
      memory: 2Gi
    requests:
      cpu: 2
      memory: 2Gi
    service:
      type: NodePort   # export fe service
      ports:
      - name: query   # fill the name from the fe service ports
        nodePort: 30930
        port: 9030
        containerPort: 9030
    storageVolumes:
    - name: fe-storage-meta
      storageClassName: rook-cephfs  # you can remove this line if you want to use the default storage class
      storageSize: 10Gi   # the size of storage volume for metadata
      mountPath: /opt/starrocks/fe/meta   # the path of metadata
    - name: fe-storage-log
      storageClassName: rook-cephfs  # you can remove this line if you want to use the default storage class
      storageSize: 5Gi    # the size of storage volume for log
      mountPath: /opt/starrocks/fe/log    # the path of log
    configMapInfo:
      configMapName: starrockscluster-fe-cm
      resolveKey: fe.conf
  starRocksCnSpec:
    image: starrocks/cn-ubuntu:latest
    replicas: 4
    limits:
      cpu: 2
      memory: 2Gi
    requests:
      cpu: 2
      memory: 2Gi
    configMapInfo:
      configMapName: starrockscluster-cn-cm
      resolveKey: cn.conf
    storageVolumes:
    - name: cn-storage-data
      storageClassName: rook-cephfs  # you can remove this line if you want to use the default storage class
      storageSize: 10Gi   # the size of storage volume for data
      mountPath: /opt/starrocks/cn/storage  # the path of data
    - name: cn-storage-log
      storageClassName: rook-cephfs  # you can remove this line if you want to use the default storage class
      storageSize: 5Gi  # the size of storage volume for log
      mountPath: /opt/starrocks/cn/log  # the path of log

```

```yaml
root@k8s-master71u:~/starrocks# cat starrockscluster-fe-cm.yaml

apiVersion: v1
kind: ConfigMap
metadata:
  name: starrockscluster-fe-cm
  namespace: starrocks
  labels:
    cluster: starrockscluster
data:
  fe.conf: |
    LOG_DIR = ${STARROCKS_HOME}/log
    DATE = "$(date +%Y%m%d-%H%M%S)"
    JAVA_OPTS="-Dlog4j2.formatMsgNoLookups=true -Xmx8192m -XX:+UseMembar -XX:SurvivorRatio=8 -XX:MaxTenuringThreshold=7 -XX:+PrintGCDateStamps -XX:+PrintGCDetails -XX:+UseConcMarkSweepGC -XX:+UseParNewGC -XX:+CMSClassUnloadingEnabled -XX:-CMSParallelRemarkEnabled -XX:CMSInitiatingOccupancyFraction=80 -XX:SoftRefLRUPolicyMSPerMB=0 -Xloggc:${LOG_DIR}/fe.gc.log.$DATE"
    JAVA_OPTS_FOR_JDK_9="-Dlog4j2.formatMsgNoLookups=true -Xmx8192m -XX:SurvivorRatio=8 -XX:MaxTenuringThreshold=7 -XX:+CMSClassUnloadingEnabled -XX:-CMSParallelRemarkEnabled -XX:CMSInitiatingOccupancyFraction=80 -XX:SoftRefLRUPolicyMSPerMB=0 -Xlog:gc*:${LOG_DIR}/fe.gc.log.$DATE:time"
    JAVA_OPTS_FOR_JDK_11="-Dlog4j2.formatMsgNoLookups=true -Xmx8192m -XX:+UseG1GC -Xlog:gc*:${LOG_DIR}/fe.gc.log.$DATE:time"
    http_port = 8030
    rpc_port = 9020
    query_port = 9030
    edit_log_port = 9010
    mysql_service_nio_enabled = true
    sys_log_level = INFO
    run_mode = shared_data
    cloud_native_meta_port = 6090
    cloud_native_storage_type = S3
    # 是否允許 StarRocks 使用 FE 配置文件中指定的儲存相關屬性創建默認儲存卷
    enable_load_volume_from_conf = true
    # 例如 testbucket/subpath
    aws_s3_path = test1
    # 例如 us-east1
    aws_s3_region = us-west-1
    # 例如 http://172.26.xx.xxx:39000
    aws_s3_endpoint = https://192.168.1.75:30443
    aws_s3_access_key = minio
    aws_s3_secret_key = minio123

```

```yaml
root@k8s-master71u:~/starrocks# cat starrockscluster-cn-cm.yaml

apiVersion: v1
kind: ConfigMap
metadata:
  name: starrockscluster-cn-cm
  namespace: starrocks
  labels:
    cluster: starrockscluster
data:
  cn.conf: |
    sys_log_level = INFO
    starlet_port = 9070
    storage_root_path = ${STARROCKS_HOME}/storage
```

```bash
root@k8s-master71u:~/starrocks# kubectl apply -f starrockscluster-fe-cm.yaml
root@k8s-master71u:~/starrocks# kubectl apply -f starrockscluster-cn-cm.yaml
root@k8s-master71u:~/starrocks# kubectl create -f shared_data_mode.yaml

```

```bash
root@k8s-master71u:~/starrocks# kubectl get pod,svc -n starrocks
NAME                                           READY   STATUS    RESTARTS         AGE
pod/kube-starrocks-operator-574b859d5f-qjxcl   1/1     Running   22 (7m33s ago)   2d2h
pod/starrockscluster-cn-0                      1/1     Running   0                49s
pod/starrockscluster-cn-1                      1/1     Running   0                49s
pod/starrockscluster-cn-2                      1/1     Running   0                49s
pod/starrockscluster-cn-3                      1/1     Running   0                49s
pod/starrockscluster-fe-0                      1/1     Running   0                98s
pod/starrockscluster-fe-1                      1/1     Running   0                98s
pod/starrockscluster-fe-2                      1/1     Running   0                98s

NAME                                  TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                                                       AGE
service/starrockscluster-cn-search    ClusterIP   None             <none>        9050/TCP                                                      49s
service/starrockscluster-cn-service   ClusterIP   10.202.106.126   <none>        9060/TCP,8040/TCP,9050/TCP,8060/TCP                           49s
service/starrockscluster-fe-search    ClusterIP   None             <none>        9030/TCP                                                      98s
service/starrockscluster-fe-service   NodePort    10.202.237.248   <none>        8030:32396/TCP,9020:32130/TCP,9030:30930/TCP,9010:32088/TCP   98s
```

```bash
root@k8s-master71u:~/starrocks# mysql -h 10.202.237.248 -P 9030 -uroot
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 2
Server version: 5.1.0 3.2.6-2585333

Copyright (c) 2000, 2024, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql>

```

指令：
https://docs.starrocks.io/zh/docs/sql-reference/sql-statements/Administration/SHOW_PROC/

查看FE狀態
```bash
# 查看FE狀態
mysql> SHOW PROC '/frontends'\G
*************************** 1. row ***************************
             Name: starrockscluster-fe-0.starrockscluster-fe-search.starrocks.svc.cluster.local_9010_1714579356541
               IP: starrockscluster-fe-0.starrockscluster-fe-search.starrocks.svc.cluster.local
      EditLogPort: 9010
         HttpPort: 8030
        QueryPort: 9030
          RpcPort: 9020
             Role: LEADER
        ClusterId: 1089252514
             Join: true
            Alive: true
ReplayedJournalId: 47
    LastHeartbeat: 2024-05-01 16:04:24
         IsHelper: true
           ErrMsg:
        StartTime: 2024-05-01 16:02:45
          Version: 3.2.6-2585333
*************************** 2. row ***************************
             Name: starrockscluster-fe-2.starrockscluster-fe-search.starrocks.svc.cluster.local_9010_1714579369126
               IP: starrockscluster-fe-2.starrockscluster-fe-search.starrocks.svc.cluster.local
      EditLogPort: 9010
         HttpPort: 8030
        QueryPort: 9030
          RpcPort: 9020
             Role: FOLLOWER
        ClusterId: 1089252514
             Join: true
            Alive: true
ReplayedJournalId: 47
    LastHeartbeat: 2024-05-01 16:04:24
         IsHelper: true
           ErrMsg:
        StartTime: 2024-05-01 16:03:02
          Version: 3.2.6-2585333
*************************** 3. row ***************************
             Name: starrockscluster-fe-1.starrockscluster-fe-search.starrocks.svc.cluster.local_9010_1714579369092
               IP: starrockscluster-fe-1.starrockscluster-fe-search.starrocks.svc.cluster.local
      EditLogPort: 9010
         HttpPort: 8030
        QueryPort: 9030
          RpcPort: 9020
             Role: FOLLOWER
        ClusterId: 1089252514
             Join: true
            Alive: true
ReplayedJournalId: 49
    LastHeartbeat: 2024-05-01 16:04:24
         IsHelper: true
           ErrMsg:
        StartTime: 2024-05-01 16:03:01
          Version: 3.2.6-2585333
3 rows in set (0.09 sec)
```

查看CN狀態
```bash
# 查看CN狀態
mysql> SHOW PROC '/compute_nodes'\G
*************************** 1. row ***************************
        ComputeNodeId: 10006
                   IP: starrockscluster-cn-0.starrockscluster-cn-search.starrocks.svc.cluster.local
        HeartbeatPort: 9050
               BePort: 9060
             HttpPort: 8040
             BrpcPort: 8060
        LastStartTime: 2024-05-01 16:03:14
        LastHeartbeat: 2024-05-01 16:03:14
                Alive: true
 SystemDecommissioned: false
ClusterDecommissioned: false
               ErrMsg:
              Version: 3.2.6-2585333
             CpuCores: 2
    NumRunningQueries: 0
           MemUsedPct: 0.00 %
           CpuUsedPct: 0.0 %
       HasStoragePath: false
          StarletPort: 9070
             WorkerId: 2
            TabletNum: 0
*************************** 2. row ***************************
        ComputeNodeId: 10004
                   IP: starrockscluster-cn-1.starrockscluster-cn-search.starrocks.svc.cluster.local
        HeartbeatPort: 9050
               BePort: 9060
             HttpPort: 8040
             BrpcPort: 8060
        LastStartTime: 2024-05-01 16:02:59
        LastHeartbeat: 2024-05-01 16:02:59
                Alive: true
 SystemDecommissioned: false
ClusterDecommissioned: false
               ErrMsg:
              Version: 3.2.6-2585333
             CpuCores: 2
    NumRunningQueries: 0
           MemUsedPct: 0.00 %
           CpuUsedPct: 0.0 %
       HasStoragePath: false
          StarletPort: 9070
             WorkerId: 1
            TabletNum: 0
*************************** 3. row ***************************
        ComputeNodeId: 10007
                   IP: starrockscluster-cn-2.starrockscluster-cn-search.starrocks.svc.cluster.local
        HeartbeatPort: 9050
               BePort: 9060
             HttpPort: 8040
             BrpcPort: 8060
        LastStartTime: 2024-05-01 16:03:24
        LastHeartbeat: 2024-05-01 16:03:24
                Alive: true
 SystemDecommissioned: false
ClusterDecommissioned: false
               ErrMsg:
              Version: 3.2.6-2585333
             CpuCores: 2
    NumRunningQueries: 0
           MemUsedPct: 0.00 %
           CpuUsedPct: 0.0 %
       HasStoragePath: false
          StarletPort: 9070
             WorkerId: 4
            TabletNum: 0
*************************** 4. row ***************************
        ComputeNodeId: 10005
                   IP: starrockscluster-cn-3.starrockscluster-cn-search.starrocks.svc.cluster.local
        HeartbeatPort: 9050
               BePort: 9060
             HttpPort: 8040
             BrpcPort: 8060
        LastStartTime: 2024-05-01 16:03:24
        LastHeartbeat: 2024-05-01 16:03:24
                Alive: true
 SystemDecommissioned: false
ClusterDecommissioned: false
               ErrMsg:
              Version: 3.2.6-2585333
             CpuCores: 2
    NumRunningQueries: 0
           MemUsedPct: 0.00 %
           CpuUsedPct: 0.0 %
       HasStoragePath: false
          StarletPort: 9070
             WorkerId: 3
            TabletNum: 0
4 rows in set (0.13 sec)
```

```bash
mysql> SHOW PROC '/jobs';
+-------+--------------------+
| DbId  | DbName             |
+-------+--------------------+
| 1     | information_schema |
| 10008 | _statistics_       |
| 100   | sys                |
+-------+--------------------+
3 rows in set (0.00 sec)

mysql> SHOW PROC '/jobs/10008';
+---------------+---------+---------+----------+-----------+-------+
| JobType       | Pending | Running | Finished | Cancelled | Total |
+---------------+---------+---------+----------+-----------+-------+
| load          | 0       | 0       | 0        | 0         | 0     |
| rollup        | 0       | 0       | 0        | 0         | 0     |
| schema_change | 0       | 0       | 0        | 0         | 0     |
| export        | 0       | 0       | 0        | 0         | 0     |
+---------------+---------+---------+----------+-----------+-------+
4 rows in set (0.03 sec)
```

指令：
https://docs.starrocks.io/zh/docs/sql-reference/sql-statements/Administration/SHOW_STORAGE_VOLUMES/
```bash
mysql> SHOW STORAGE VOLUMES;
+------------------------+
| Storage Volume         |
+------------------------+
| builtin_storage_volume |
+------------------------+
1 row in set (0.07 sec)
```

```bash
mysql> CREATE STORAGE VOLUME def_volume
    -> TYPE = S3
    -> LOCATIONS = ("s3://test1/")
    -> PROPERTIES
    -> (
    ->     "enabled" = "true",
    ->     "aws.s3.region" = "us-west-1",
    ->     "aws.s3.endpoint" = "https://192.168.1.75:30443",
    ->     "aws.s3.access_key" = "minio",
    ->     "aws.s3.secret_key" = "minio123",
    ->     "aws.s3.enable_partitioned_prefix" = "true"
    -> );
Query OK, 0 rows affected (0.07 sec)


mysql> SET def_volume AS DEFAULT STORAGE VOLUME;
Query OK, 0 rows affected (0.04 sec)


mysql> CREATE DATABASE cloud_db;
Query OK, 0 rows affected (0.05 sec)


mysql> USE cloud_db;
Database changed
mysql> CREATE TABLE IF NOT EXISTS detail_demo (
    ->     recruit_date  DATE           NOT NULL COMMENT "YYYY-MM-DD",
    ->     region_num    TINYINT        COMMENT "range [-128, 127]",
    ->     num_plate     SMALLINT       COMMENT "range [-32768, 32767]")
    ->
    -> DUPLICATE KEY(recruit_date, region_num)
    -> DISTRIBUTED BY HASH(recruit_date, region_num)
    -> PROPERTIES (
    ->     "storage_volume" = "def_volume",
    ->     "datacache.enable" = "true",
    ->     "datacache.partition_duration" = "1 MONTH",
    ->     "enable_async_write_back" = "false"
    -> );
ERROR 1064 (HY000): Unexpected exception: fail to create tablet: 10006: [Internal error: starlet err [RequestID=0][StatusCode=400]Put object s3://test1/11/dab022f1-c620-4be6-909d-2b5ce00d9554/db10243/10258/10257/meta/0000000000002817_0000000000000001.meta error: Unable to parse ExceptionName: InvalidArgument Message: S3 API Requests must be made to API port.]
```

