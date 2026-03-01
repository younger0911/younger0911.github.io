---
title: fabric8io kunernetes client 通过token建立访问
date: 2021-09-26 14:15:04
tags:
	- kubernetes
	- kubernetes-client
categories:
	- 后端
	- 容器
---



### 一、 获取token：

#### 查看账号

`kubectl -n kube-system get sa`

<!-- more -->

![image-20200518155520647](/images/image-20200518155520647.png)

#### 按照账号取得secret

`kubectl -n kube-system get sa admin-metrices -o yaml`

![image-20200518155713836](/images/image-20200518155713836.png)

#### 取得token并解码

`kubectl get secrets admin-metrices-token-mszpc -n kube-system -o yaml`

![image-20200518155903430](/images/image-20200518155903430.png)

`kubectl get secret admin-metrices-token-mszpc -n kube-system -o jsonpath={".data.token"} | base64 -d`

![image-20200518160106736](/images/image-20200518160106736.png)

#### 验证token是否有效

`curl -k -H 'Authorization: Bearer ${token}' https://${ip}:${port}/api`

![image-20200518160353836](/images/image-20200518160353836.png)



---

或者利用rbac脚本创建一个账号：

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: scheduler-metrics-reader  #建立账号
  namespace: kube-system

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole  # 集群角色
metadata:  # 此处的 "namespace" 被省略掉是因为 ClusterRoles 是没有命名空间的。
  name: metrics-reader
rules:
  - apiGroups: ["", "metrics.k8s.io"] # "" 指定核心 API 组；"metrics.k8s.io" 获取metrics
    resources: ["nodes", "pods"]  # 资源类型
    verbs: ["get", "watch", "list"]  #权限

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding # 此角色绑定使得用户 "scheduler-node-reader" 能够读取 "vcs" 命名空间中的 nodes
metadata:
  name: read-nodes
  namespace: vcs
subjects:
  - kind: ServiceAccount
    name: scheduler-metrics-reader # Name is case sensitive
    namespace: kube-system
roleRef:
  kind: ClusterRole #this must be Role or ClusterRole
  name: metrics-reader # 这里的名称必须与你想要绑定的 Role 或 ClusterRole 名称一致
  apiGroup: rbac.authorization.k8s.io
```

执行：`kubectl apply -f rbac.yaml`

利用shell脚本一键获取token：

```shell
#!/bin/bash

info=$(sudo kubectl cluster-info)
temp=${info#*at}
url=${temp%CoreDNS*}
echo "$url"

name=$(sudo kubectl -n kube-system get sa scheduler-metrics-reader -o jsonpath={".secrets[0].name"})
token=$(sudo kubectl get secret "${name}" -n kube-system -o jsonpath={".data.token"} | base64 -d)
echo "$token"
```



### 二、 访问集群API：

#### 使用`https://github.com/fabric8io/kubernetes-client`访问k8s

```java
ConfigBuilder builder = new ConfigBuilder();
builder.withTrustCerts(true);
builder.withMasterUrl("https://localhost:6443");
builder.withOauthToken(token);  //解码后的token，不带Bearer
Config config = builder.build();
final KubernetesClient client = new DefaultKubernetesClient(config);
```

* 附：

maven依赖(具体版本看git上的Compatibility Matrix)

```yaml
<!-- kubernetes依赖 -->
<dependency>
	<groupId>io.fabric8</groupId>
	<artifactId>kubernetes-client</artifactId>
	<version>${kubernetes.version}</version>
</dependency>
<dependency>
	<groupId>io.fabric8</groupId>
	<artifactId>kubernetes-model</artifactId>
	<version>${kubernetes.version}</version>
</dependency>
```

* 附2：

按照pod的特定属性获取指定pod，例如根据pod的nodeName获取pod：

![image-20200609162122310](/images/image-20200609162122310.png)

可以看到nodeName属性在pod的spec下，则找出该node下pod的列表（这里k8sclient根据上述生成，可以在springboot启动时生成到bean中）：

```java
PodList pods = k8sClient.pods().withField("spec.nodeName", "192.168.0.152").list();
```



参考
[k8s使用ServiceAccount Token的方式访问apiserver](https://yq.aliyun.com/articles/672460)

