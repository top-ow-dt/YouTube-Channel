
k8s使用nfs作為動態storageClass儲存

部署 StarRocks Operator
添加訂製資源 StarRocksCluster
```bash

kubectl apply -f https://raw.githubusercontent.com/StarRocks/starrocks-kubernetes-operator/main/deploy/starrocks.com_starrocksclusters.yaml
```
部署 StarRocks Operator
```bash
kubectl apply -f https://raw.githubusercontent.com/StarRocks/starrocks-kubernetes-operator/main/deploy/operator.yaml
```
部署 StarRocks 集群
配置文件範例

我這裡配置一個使用騰訊雲COS做存算分離+storageClass做FE元數據持久化

shared_data_mode.yaml
```yaml
apiVersion: starrocks.com/v1
kind: StarRocksCluster
metadata:
  name: starrockscluster
  namespace: starrocks
spec:
  starRocksFeSpec:
    image: starrocks/fe-ubuntu:3.2.2
    replicas: 1
    limits:
      cpu: 2
      memory: 4Gi
    requests:
      cpu: 2
      memory: 4Gi
    # service:
    #   type: NodePort   # export fe service
    #   ports:
    #   - name: query   # fill the name from the fe service ports
    #     nodePort: 32755
    #     port: 9030
    #     containerPort: 9030
    storageVolumes:
    - name: fe-storage-meta
      storageClassName: "nfs-client"  # you can remove this line if you want to use the default storage class
      storageSize: 10Gi   # the size of storage volume for metadata
      mountPath: /opt/starrocks/fe/meta   # the path of metadata
    - name: fe-storage-log
      storageClassName: "nfs-client"  # you can remove this line if you want to use the default storage class
      storageSize: 1Gi    # the size of storage volume for log
      mountPath: /opt/starrocks/fe/log    # the path of log
    configMapInfo:
      configMapName: starrockscluster-fe-cm
      resolveKey: fe.conf
  starRocksCnSpec:
    image: starrocks/cn-ubuntu:3.2.2
    replicas: 1
    limits:
      cpu: 2
      memory: 4Gi
    requests:
      cpu: 2
      memory: 4Gi
    configMapInfo:
      configMapName: starrockscluster-cn-cm
      resolveKey: cn.conf
    storageVolumes:
    - name: cn-storage-data
      storageClassName: "nfs-client"  # you can remove this line if you want to use the default storage class
      storageSize: 10Gi   # the size of storage volume for data
      mountPath: /opt/starrocks/cn/storage  # the path of data
    - name: cn-storage-log
      storageClassName: "nfs-client"  # you can remove this line if you want to use the default storage class
      storageSize: 1Gi  # the size of storage volume for log
      mountPath: /opt/starrocks/cn/log  # the path of log
  starRocksFeProxySpec:
    replicas: 1
    limits:
      cpu: 1
      memory: 2Gi
    requests:
      cpu: 1
      memory: 2Gi
    service:
      type: NodePort   # export fe proxy service
      ports:
        - name: http-port   # fill the name from the fe proxy service ports
          containerPort: 8080
          nodePort: 30180   # The range of valid ports is 30000-32767
          port: 8080
    resolver: "kube-dns.kube-system.svc.cluster.local"  # this is the default dns server.

---

# fe config
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
    # 是否允許 StarRocks 使用 FE 配置文件中指定的儲存相關屬性創建默認儲存卷
    enable_load_volume_from_conf = true
    aws_s3_path = files-prod-1253767413/starrocks
    aws_s3_region = ap-guangzhou
    aws_s3_endpoint = https://cos.ap-guangzhou.myqcloud.com
    aws_s3_access_key = XXX
    aws_s3_secret_key = XXX

---

# cn config
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
    # ports for admin, web, heartbeat service
    thrift_port = 9060
    webserver_port = 8040
    heartbeat_service_port = 9050
    brpc_port = 8060
```

```bash
kubectl apply -f shared_data_mode.yaml
```
