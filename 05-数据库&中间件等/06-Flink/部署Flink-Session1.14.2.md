# Flink Session高可用部署

## 部署信息

- 官方项目地址：[Flink-Session-HA](https://nightlies.apache.org/flink/flink-docs-release-1.14/zh/docs/deployment/resource-providers/standalone/kubernetes/)
- Flink版本：1.14.2
- 部署架构：1×Jobmanager + 2×Taskmanager，通过持久化recovery,checkpoints实现故障恢复
- Prometheus监控集成

## 部署步骤

### Namespace

```shell
kubectl create namespace flink-session
```

### RBAC

```shell
kubectl create serviceaccount flink-service-account -n flink-session
kubectl create clusterrolebinding flink-role-binding-flink \
    --clusterrole=edit \
    --serviceaccount=flink-session:flink-service-account
```

### PV&PVC

- 提前创建目录并设置权限

```shell
# 进入nfs节点的挂载目录
cd /data/nfs
mkdir -p flink-session/{upload,recovery,checkpoints}
chown -R 9999:9999 flink-session/
```

- 创建PV&PVC

```shell
vim 1-flink-static-pv_pvc.yaml
```

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: flink-session-pv
spec:
  storageClassName: "flink-session"
  capacity:
    storage: 10Gi
  accessModes:
  - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  nfs:
    path: /data/nfs/flink-session
    server: master01
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: flink-session-pvc
  namespace: flink-session
spec:
  storageClassName: "flink-session"
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
```

### Configmap

```shell
vim 2-flink-configuration-configmap.yaml
```

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: flink-config
  namespace: flink-session
  labels:
    app: flink-session
data:
  flink-conf.yaml: |+
    kubernetes.cluster-id: 1338
    high-availability: org.apache.flink.kubernetes.highavailability.KubernetesHaServicesFactory
    high-availability.storageDir: file:///opt/flink/recovery
    restart-strategy: fixed-delay
    restart-strategy.fixed-delay.attempts: 10
    jobmanager.web.upload.dir: /opt/flink/upload
    jobmanager.rpc.address: flink-jobmanager
    taskmanager.numberOfTaskSlots: 2
    blob.server.port: 6124
    jobmanager.rpc.port: 6123
    taskmanager.rpc.port: 6122
    queryable-state.proxy.ports: 6125
    jobmanager.memory.process.size: 1600m
    taskmanager.memory.process.size: 1728m
    parallelism.default: 2 
    state.backend: filesystem
    state.checkpoints.dir: file:///opt/flink/checkpoints
    execution.checkpointing.interval: 10s
  log4j-console.properties: |+
    # 如下配置会同时影响用户代码和 Flink 的日志行为
    rootLogger.level = INFO
    rootLogger.appenderRef.console.ref = ConsoleAppender
    rootLogger.appenderRef.rolling.ref = RollingFileAppender

    # 如果你只想改变 Flink 的日志行为则可以取消如下的注释部分
    #logger.flink.name = org.apache.flink
    #logger.flink.level = INFO

    # 下面几行将公共 libraries 或 connectors 的日志级别保持在 INFO 级别。
    # root logger 的配置不会覆盖此处配置。
    # 你必须手动修改这里的日志级别。
    logger.akka.name = akka
    logger.akka.level = INFO
    logger.kafka.name= org.apache.kafka
    logger.kafka.level = INFO
    logger.hadoop.name = org.apache.hadoop
    logger.hadoop.level = INFO
    logger.zookeeper.name = org.apache.zookeeper
    logger.zookeeper.level = INFO

    # 将所有 info 级别的日志输出到 console
    appender.console.name = ConsoleAppender
    appender.console.type = CONSOLE
    appender.console.layout.type = PatternLayout
    appender.console.layout.pattern = %d{yyyy-MM-dd HH:mm:ss,SSS} %-5p %-60c %x - %m%n

    # 将所有 info 级别的日志输出到指定的 rolling file
    appender.rolling.name = RollingFileAppender
    appender.rolling.type = RollingFile
    appender.rolling.append = false
    appender.rolling.fileName = ${sys:log.file}
    appender.rolling.filePattern = ${sys:log.file}.%i
    appender.rolling.layout.type = PatternLayout
    appender.rolling.layout.pattern = %d{yyyy-MM-dd HH:mm:ss,SSS} %-5p %-60c %x - %m%n
    appender.rolling.policies.type = Policies
    appender.rolling.policies.size.type = SizeBasedTriggeringPolicy
    appender.rolling.policies.size.size=100MB
    appender.rolling.strategy.type = DefaultRolloverStrategy
    appender.rolling.strategy.max = 10

    # 关闭 Netty channel handler 中不相关的（错误）警告
    logger.netty.name = org.jboss.netty.channel.DefaultChannelPipeline
    logger.netty.level = OFF  
```

### Jobmanager

```shell
vim 3-jobmanager-session-deployment-ha.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flink-jobmanager
  namespace: flink-session
spec:
  replicas: 1 # 通过设置大于 1 的整型值来开启 Standby JobManager
  selector:
    matchLabels:
      app: flink-session
      component: jobmanager
  template:
    metadata:
      labels:
        app: flink-session
        component: jobmanager
    spec:
      containers:
      - name: jobmanager
        image: apache/flink:1.14.2-scala_2.11
        env:
        - name: POD_IP
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: status.podIP
        # 下面的 args 参数会使用 POD_IP 对应的值覆盖 config map 中 jobmanager.rpc.address 的属性值。
        args: ["jobmanager", "$(POD_IP)"]
        ports:
        - containerPort: 6123
          name: rpc
        - containerPort: 6124
          name: blob-server
        - containerPort: 8081
          name: webui
        livenessProbe:
          tcpSocket:
            port: 6123
          initialDelaySeconds: 30
          periodSeconds: 60
        volumeMounts:
        - name: flink-config-volume
          mountPath: /opt/flink/conf
        - name: flink-pvc
          subPath: recovery
          mountPath: /opt/flink/recovery
        - name: flink-pvc
          subPath: upload
          mountPath: /opt/flink/upload
        - name: flink-pvc
          subPath: checkpoints
          mountPath: /opt/flink/checkpoints
        securityContext:
          runAsUser: 9999  # 参考官方 flink 镜像中的 _flink_ 用户，如有必要可以修改
      serviceAccountName: flink-service-account # 拥有创建、编辑、删除 ConfigMap 权限的 Service 账号
      volumes:
      - name: flink-config-volume
        configMap:
          name: flink-config
          items:
          - key: flink-conf.yaml
            path: flink-conf.yaml
          - key: log4j-console.properties
            path: log4j-console.properties
      - name: flink-pvc
        persistentVolumeClaim:
          claimName: flink-static-nas-pvc
```

### Taskmanager

```shell
vim 4-taskmanager-session-deployment.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flink-taskmanager
  namespace: flink-session
spec:
  replicas: 2
  selector:
    matchLabels:
      app: flink-session
      component: taskmanager
  template:
    metadata:
      labels:
        app: flink-session
        component: taskmanager
    spec:
      containers:
      - name: taskmanager
        image: apache/flink:1.14.2-scala_2.11
        args: ["taskmanager"]
        ports:
        - containerPort: 6122
          name: rpc
        - containerPort: 6125
          name: query-state
        livenessProbe:
          tcpSocket:
            port: 6122
          initialDelaySeconds: 30
          periodSeconds: 60
        volumeMounts:
        - name: flink-config-volume
          mountPath: /opt/flink/conf/
        securityContext:
          runAsUser: 9999  # 参考官方 flink 镜像中的 _flink_ 用户，如有必要可以修改
      serviceAccountName: flink-service-account
      volumes:
      - name: flink-config-volume
        configMap:
          name: flink-config
          items:
          - key: flink-conf.yaml
            path: flink-conf.yaml
          - key: log4j-console.properties
            path: log4j-console.properties
```

```shell
vim 5-jobmanager-rest-service.yaml 
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: flink-jobmanager-rest
  namespace: flink-session
spec:
  type: NodePort
  ports:
  - name: rest
    port: 8081
    targetPort: 8081
    nodePort: 30081
  selector:
    app: flink-session
    component: jobmanager
```



