# kubernetes配置与密钥管理

## ConfigMap

### configmap介绍

kubernetes 集群可以使用ConfigMap实现对容器中赢得配置管理，可以把configmap看成是挂载到pod中的存储卷。
- 明文保存
- 常用于配置文件管理


### ConfigMap创建的四种方式

#### 1.在命令行中指定参数创建

``` 
~]# kubectl create configmap cm1 --from-literal=host=127.0.0.1 --from-literal=port=3306
configmap/cm1 created
```
``` 
~]# kubectl  describe cm/cm1
Name:         cm1
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
host:
----
127.0.0.1
port:
----
3306

BinaryData
====

Events:  <none> 
```

#### 2.在命令行通过多个文件创建

```
~]# echo -n 127.0.0.1 > host
~]# echo -n 3306 > port
~]# kubectl  create cm cm2 --from-file=./host --from-file=./port
~]# kubectl  describe cm/cm2
Name:         cm2
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
host:
----
127.0.0.1
port:
----
3306

BinaryData
====

Events:  <none>
```

#### 3.在命令行通过文件提供多个键值对创建
```
~]# cat env.txt
host=127.0.0.1
port=3306
~]# kubectl  create configmap cm3 --from-env-file=env.txt
configmap/cm3 created
~]# kubectl  describe cm/cm3
Name:         cm3
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
host:
----
127.0.0.1
port:
----
3306

BinaryData
====

Events:  <none>
```

#### 4.通过YAML资源清单创建
```
cat cm4.yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: cm4
data:
  host: 127.0.0.1
  port: "3306"
  
~]# kubectl  apply -f cm4.yaml
configmap/cm4 created
[root@k8s-master-01 ~]# kubectl  describe cm/cm4
Name:         cm4
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
host:
----
127.0.0.1
port:
----
3306

BinaryData
====

Events:  <none>
```

### configmap的二种使用方式

#### 1.通过环境变量的方式传递给Pod

通过环境变量传递给Pod，这种方式不支持热部署

``` 
~]# cat cm-pod-env.yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: cm-pod-env
spec:
  containers:
  - name: busybox
    image: busybox:1.28
    args: ["/bin/sh", "-c", "sleep 3600"]
    envFrom:
    - configMapRef:
        name: cm1
~]# kubectl  apply -f cm-pod-env.yaml
~]# kubectl  exec -it cm-pod-env -- env
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=cm-pod-env
host=127.0.0.1
port=3306
...
```

#### 2.通过volumes的方式挂载到Pod

通过volumes的方式挂载到Pod，key为挂载目录下的文件名，value为文件名内的内容，此方式**支持热更新**，更新时间间隔大概在30s左右。
``` 
~]# cat cm-pod-vol.yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: cm-pod-vol
spec:
  containers:
  - name: busybox
    image: busybox:1.28
    args: ["/bin/sh", "-c", "sleep 3600"]
    volumeMounts:
    - name: vol-cm
      mountPath: "/data/mysql/etc"
      readOnly: true
  volumes:
  - name: vol-cm
    configMap:
      name: cm2
      
~]# kubectl apply -f  cm-pod-vol.yaml
~]# kubectl  exec -it cm-pod-vol -- ls /data/mysql/etc/
host  port
~]# kubectl  exec -it cm-pod-vol -- cat /data/mysql/etc/host
127.0.0.1
```

## Secret
### secret介绍

Secret与configmap类似，主要区别是ConfigMap是明文，而Secret存储的base64加密后的密文
- base64加密的密文，可通过`base64 -d`进行解密
- 常用于密码、密钥、token等敏感数据的配置管理

### secret类型

Secret有4中类型
- Opaque: base64编码格式的Secret，用来存储密码、密钥、信息、证书等，类型标识符为generic
- Service Account： 用来访问kubernetes api，由kubernetes自动创建，并且自动挂载到Pod的/var/run/secrets/kubernetes.io/serviceaccount目录
- kubernetes.io/dockerconfigjson: 用来存储docker register的认证信息，类型标识符为 docker-register
- kubernetes/tls： 用于为SSL通信模式存储证书合私钥文件，类型标识符为tls
```
~]# kubectl  create secret -h
Create a secret with specified type.

 A docker-registry type secret is for accessing a container registry.

 A generic type secret indicate an Opaque secret type.

 A tls type secret holds TLS certificate and its associated key.

Available Commands:
  docker-registry   创建一个给 Docker registry 使用的 Secret
  generic           Create a secret from a local file, directory, or literal value
  tls               创建一个 TLS secret

Usage:
  kubectl create secret (docker-registry | generic | tls) [options]

Use "kubectl create secret <command> --help" for more information about a given command.
Use "kubectl options" for a list of global command-line options (applies to all commands).
```

### secret应用案例

#### 数据库密码案例

创建数据库管理员密码Secret案例，使用Opaque类型来创建mysql密码Secret

将明文进行base64编码
```
# 假设数据库密码为root1234 ,这里一定要使用 -n 选项，否则mysql pod会创建失败
~]# echo -n root1234|base64
cm9vdDEyMzQ=
```
2.编写YAML资源清单文件
``` 
~]# cat secret-mysql.yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: secret-mysql
data:
  password: cm9vdDEyMzQ=
~]# kubectl  apply -f mysql-secret.yaml
secret/secret-mysql created

~]# kubectl  describe secret/secret-mysql
Name:         secret-mysql
Namespace:    default
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
passowrd:  10 bytes
```

### secret的2种使用方式
- env：环境变量方式不支持热更新
- volume: 卷挂载方式支持热更新

#### 1.通过环境变量方式传递给Pod



1.编写资源清单文件使用secret-mysql
``` 
~]# cat secret-mysql-env.yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: secret-mysql-env
spec:
  containers:
  - name: mysql
    image: mysql:5.7
    env:
      - name: MYSQL_ROOT_PASSWORD
        valueFrom:
          secretKeyRef:
            name: secret-mysql
            key: password
~]# kubectl  apply -f secret-mysql-env.yaml
pod/secret-pod-env created
```
2.验证pod
```
~]# kubectl  exec -it pod/secret-mysql-env -- mysql -uroot
ERROR 1045 (28000): Access denied for user 'root'@'localhost' (using password: NO)
command terminated with exit code 1
~]# kubectl  exec -it pod/secret-mysql-env -- mysql -uroot  -proot1234
...
Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql>
```
#### 2.通过volume的方式挂载到Pod

1.编写资源清单文件使用secret-mysql
``` 
~]# cat secret-pod-vol.yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: secret-pod-vol
spec:
  containers:
  - name: busybox
    image: busybox:1.20
    args:
    - /bin/sh
    - -c
    - sleep 3600
    volumeMounts:
    - name: vol-secret
      mountPath: "/opt/password" # 挂载目录(支持热更新)，也可以使用subPath(不支持热更新)
      readOnly: true
  volumes:
  - name: vol-secret
    secret:
      secretName: secret-mysql
[root@k8s-master-01 ~]# kubectl  apply -f secret-pod-vol.yaml
pod/secret-pod-vol created
```