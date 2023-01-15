### 为什么需要pod
pod是kubernetes的原子调度单位。

- 能更好的处理具有超亲密关系的一组容器。
- 容器设计模式
   - 共享本地网络和同一个volume
在单容器模式中，如果A B想要共享网络栈和volume，那么就要使用类似 `docker run --net --volume-from`这样的命令 这样 A B两个容器就有先后顺序 是拓扑关系。而在pod中，pod的实现需要使用一个中间容器 `Infra`。如果要使用pod 那么就必须先创建`infra`容器。用户定义的其他容器则通过 Join Network Namespace 的方式与 infra 容器关联在一起![image.png](https://cdn.nlark.com/yuque/0/2023/png/415797/1673589309514-6492f6dd-124c-4ea6-a10a-2c1e4db49f45.png#averageHue=%236c9cca&clientId=uef84e86e-4968-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=518&id=u9a887947&margin=%5Bobject%20Object%5D&name=image.png&originHeight=518&originWidth=577&originalType=binary&ratio=1&rotation=0&showTitle=false&size=94987&status=done&style=none&taskId=ucd7e4355-a6e0-4ff2-9114-420a1db1085&title=&width=577)

在kubernetes中这个镜像就是 pause 镜像。在同一个pod中的容器流量进出都通过infra容器完成的。
共享volume则是通过在pod层定义volume。pod里面的容器只要声明挂载就可以了。
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: two-containers
spec:
  restartPolicy: Never
  volumes:
  - name: shared-data
    hostPath:
      path: /data
  containers:
  - name: nginx-container
    image: nginx
    volumeMounts:
    - name: shared-data
      mountPath: /usr/share/nginx/html
  - name: debian-container
    image: debian
    volumeMounts:
    - name: shared-data
      mountPath: /pod-data
    command: ["/bin/sh"]
    args: ["-c", "echo Hello from the debian container > /pod-data/index.html"]

```

   - 边车模式

通过可以共享网络和volume，可以实现边车模式。比如将应用容器和工具容器解耦。
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: javaweb-2
spec:
  initContainers:
  - image: geektime/sample:v2
    name: war
    command: ["cp", "/sample.war", "/app"]
    volumeMounts:
    - mountPath: /app
      name: app-volume
  containers:
  - image: geektime/tomcat:7.0
    name: tomcat
    command: ["sh","-c","/root/apache-tomcat-7.0.42-v2/bin/start.sh"]
    volumeMounts:
    - mountPath: /root/apache-tomcat-7.0.42-v2/webapps
      name: app-volume
    ports:
    - containerPort: 8080
      hostPort: 8001
  volumes:
  - name: app-volume
    emptyDir: {}
```
### 深入解析pod对象
pod更像传统部署模式中的虚拟机 而容器更像是用户程序。所以可以在pod的属性中定义：调度、网络、存储以及安全相关的属性。

- NodeSelector

将pod和node绑定的字段
```yaml
spec:
  nodeSelector:
    disktype: ssd  # 只会将这个pod调度到 含有 disktype = ssd 的node上
```

- hostAlias

注入到pod中容器的 /etc/hosts 文件中
```yaml
spec:
  hostAliases:
  - ip: "127.0.0.1"
    hostnames:
    - "foo.local"
    - "bar.local"
```

- shareProcessNamespace

pod中所有的容器共享PID namespace。其中 infra 容器的pid为1

- initContainers
在启动容器之前 启动的一组初始化容器。
- containers
pod中启动的容器
   - env
设置的环境变量
   - envFrom
从configMap中读取环境变量
```yaml
spec:
  containers:
  - name: envar-demo-container
    image: gcr.io/google-samples/node-hello:1.0
    env:
    - name: DEMO_GREETING
      value: "Hello from the environment"
    envFrom:
    - configMapRef:
      name: special-config
```

   - imagePullPolicy
One of Always, Never, IfNotPresent. Defaults to Always。if :latest tag is specified, or IfNotPresent otherwise. Cannot be updated.
   - lifecycle
### pod对象使用进阶
#### 关于四种 Project Volume （投射数据卷）
Project Volume 中的数据是为容器提供预先定义好的数据。包括配置 密码 等信息
##### configMap
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coder
data:
  config: |
    I am a coder.
    My blog is didispace.com.

---
apiVersion: v1
kind: Pod
metadata:
  name: pod-mount-configmap
spec:
  volumes:
    - name: ccc
      configMap:
        name: coder
  containers:
    - name: alpine
      image: alpine
      command: ['sh', "-c", "sleep 3600;"]
      volumeMounts:
        - mountPath: /config
          readOnly: true
          name: ccc
```
```shell
/config # ls
config
/config # cat config
I am a coder.
My blog is didispace.com.
```
##### secret
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: zhangyu
type: Opaque
data:
  name: emhhbmd5dQ==
  secret: VUludDhBcnJheQ==

---
apiVersion: v1
kind: Pod
metadata:
  name: pod-mount-secret
spec:
  volumes:
    - name: zzz
      secret:
        secretName: zhangyu
  containers:
    - name: alpine
      image: alpine
      command: ['sh', "-c", "sleep 3600;"]
      volumeMounts:
        - mountPath: /secret
          readOnly: true
          name: zzz
```
```shell
/ # cd /secret/
/secret # ls
name    secret
/secret # cat name
```
##### downwardAPI
pod中的container可以读取pod的信息。比如pod的标签，cpu的个数等
##### ServiceAccountToken
如果想在pod中控制k8s，那么就需要有权限去访问k8s，这个ServiceAccountToken就是访问集群的凭证。
#### 正确的设置 livenessProbe
在containers 的配置中配置 livenessProbe 。来做健康检查。这样当镜像的状态不健康的时候 就会重启pod里面的容器。pod的restartPolicy属性就是在container的状态不健康的时候做的动作，Restart policy for all containers within the pod. One of Always, OnFailure, Never. Default to Always.
#### 定义 PodPreset
podpreset 就是pod的预设置。当创建的pod label和podpreset匹配的时候 就会将podpreset中预设的值merge到pod的配置中
```yaml
apiVersion: settings.k8s.io/v1alpha1
kind: PodPreset
metadata: 
    name: allow-database
spec: 
    selector: 
        matchLabels: 
            role: frontend 
    env:
        - name: DB_PORT 
          value: "6379"
    volumeMounts: 
        - mountPath: /cache 
          name: cache-volume 
    volumes: 
        - name: cache-volume 
          emptyDir: {}
```
当创建的pod含有 role = frontend 的标签的时候 就会将 preset中的值 merge到创建的pod中
```yaml
apiVersion: v1 
kind: Pod 
metadata: 
    name: website
    labels: 
        app: website 
        role: frontend 
spec: 
    containers: 
        - name: website 
          image: nginx 
          ports: 
            - containerPort: 80
```
