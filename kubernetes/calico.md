## calico 组件




## 

### 阻止来自其他namespace的访问

创建`test-web` 并创建service:

```sh
kubectl create namespace test-web

kubectl run web --namespace test-web --image=nginx \
    --labels=app=web --expose --port 80
```

测试

```sh
$ kubectl run test-$RANDOM --namespace=default --rm -i -t --image=alpine -- sh
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
$ kubectl run test-$RANDOM --namespace=default --rm -i -t --image=alpine -- sh
/ # wget -qO- --timeout=2 http://web.test-web
wget: download timed out
```

此时来自`default`namespace的访问 已经被禁止了


测试来自`test-web`namespace的访问

```sh
$ kubectl run test-$RANDOM --namespace=test-web --rm -i -t --image=alpine -- sh
/ # wget -qO- --timeout=2 http://web.test-web
<!DOCTYPE html>
<html>
```

访问OK

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
$ kubectl run test-$RANDOM --namespace=test-web --rm -i -t --image=alpine -- sh
/ # wget -qO- --timeout=2 http://web.test-web
wget: download timed out
```
访问被禁止


测试带label标签pod进行访问
```sh
$ kubectl run test-$RANDOM --namespace=test-web --labels="access=true" --rm -i -t --image=alpine -- sh
/ # wget -qO- --timeout=2 http://web.test-web
wget: download timed out
```

可以正常访问



### 清除测试环境
    kubectl delete namespace test-web
