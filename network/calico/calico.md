## calico 架构

<img src="./calico.jpg" width="500" height="400">

calico/node:
- flex：
  端口管理：端口的创建删除、包括这些端口arp通信等
  管理路由：将路由信息写入kernel路由
  ACl：将acl信息写入iptables
- bird:
  分发路由信息
- confd:
  模版服务当路由配置变更之后，动态加载路由配置信息

Etcd：
  数存储



### 阻止来自其他namespace的访问

创建`test-web` 并创建service:

```sh
kubectl create namespace test-web

kubectl run web -n test-web --image=nginx \
    --labels=app=web --expose --port 80
```

测试

```sh
$ kubectl run test-$RANDOM -n default --rm -i -t --image=alpine -- sh
/ # wget -qO- --timeout=2 http://web.test-web
<!DOCTYPE html>
<html>
```

默认是允许任何流量访问与流出的


创建`deny-from-other-namespaces.yaml` 并创建kubernetes集群NetworkPolicy资源

```yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  namespace: test-web
  name: deny-from-other-namespaces
spec:
  podSelector:
    matchLabels:
  ingress:
  - from:
    - podSelector: {}
```

```sh
$ kubectl apply -f deny-from-other-namespaces.yaml
networkpolicy "deny-from-other-namespaces" created"
```




测试来自`default`namespace的访问

```sh
$ kubectl run test-$RANDOM -n default --rm -i -t --image=alpine -- sh
/ # wget -qO- --timeout=2 http://web.test-web
wget: download timed out
```

此时来自`default`namespace的访问 已经被禁止了


测试来自`test-web`namespace的访问

```sh
$ kubectl run test-$RANDOM -n test-web --rm -i -t --image=alpine -- sh
/ # wget -qO- --timeout=2 http://web.test-web
<!DOCTYPE html>
<html>
```

访问OK



### 测试指定namespace可以访问

创建测试ns `test-dev`并打一个`test`label
```sh
kubectl create namespace test-dev
kubectl label namespace/test-dev purpose=test
```

我们设置允许这个`test-dev`namespace的流量可以访问`test-web`namespace重的web服务
添加networkpolicy
创建`allow-from-test-dev-test.yaml` 并创建kubernetes集群NetworkPolicy资源

```yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: web-allow-test-dev-test
  namespace: test-web
spec:
  podSelector:
    matchLabels:
      app: web
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          purpose: test
```
```sh
$ kubectl apply -f allow-from-test-dev-test.yaml
networkpolicy "web-allow-test-dev-test" created
```


测试从`default`namespace 访问`test-web`服务



```sh
$ kubectl run test-$RANDOM -n default --rm -i -t --image=alpine -- sh
/ # wget -qO- --timeout=2 http://web.test-web
wget: download timed out
```
访问被禁止


测试从`test-dev`namespace 访问`test-web`服务
```sh
$ kubectl run test-$RANDOM -n test-dev --rm -i -t --image=alpine -- sh
/ # wget -qO- --timeout=2 http://web.test-web
<!DOCTYPE html>
<html>
```

可以正常访问






### 测试指定pod可以访问

删除之前netpol
```sh
$ kubectl -n test-web delete netpol --all

```


创建`allow-from-access-pod.yaml` 并创建kubernetes集群NetworkPolicy资源

```yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  namespace: test-web
  name: allow-from-access-pod
spec:
  podSelector:
    matchLabels:
  ingress:
  - from:
    - podSelector:
        matchLabels:
          access: "true"
```

测试不带label访问


```sh
$ kubectl run test-$RANDOM -n test-web --rm -i -t --image=alpine -- sh
/ # wget -qO- --timeout=2 http://web.test-web
wget: download timed out
```
访问被禁止


测试带label标签pod进行访问
```sh
$ kubectl run test-$RANDOM -n test-web --labels="access=true" --rm -i -t --image=alpine -- sh
/ # wget -qO- --timeout=2 http://web.test-web
<!DOCTYPE html>
<html>
```

可以正常访问



### 清除测试环境
    kubectl delete namespace test-web
    kubectl delete namespace test-dev
