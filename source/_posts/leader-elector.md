---
title: k8s是如何选主的
categories:
- 云原生
---
本文主要介绍两种选主方式:
- Sidercar部署：不侵入业务，利用现有轮子
- 嵌入业务代码：对于业务有侵入，好在可控性强

# Sidercar部署方式
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
  labels:
    app: nginx-deploy
spec:
  replicas: 2
  selector:
    matchLabels: &labels
      app: nginx-deploy
  template:
    metadata:
      labels: *labels
    spec:
      containers:
      - args:
        - --election=elect-nginx-deploy
        - --http=0.0.0.0:4444
        image: k8s.gcr.io/leader-elector:0.5
        imagePullPolicy: IfNotPresent
        name: leader-elector
      - name: nginx-deploy
        image: nginx
        imagePullPolicy: IfNotPresent
        ports:
          - containerPort: 9898  #这里containerPort是容器内部的port, 只能在容器内部访问
```
报错如下
```
$ kubectl logs pod/nginx-deploy-84464d8b45-hnfzf -n test-leader-elector
Defaulted container "leader-elector" out of: leader-elector, nginx-deploy
F0303 14:30:00.130485       8 main.go:108] failed to create election: endpoints "elect-nginx-deploy" is forbidden: User "system:serviceaccount:test-leader-elector:default" cannot get resource "endpoints" in API group "" in the namespace "default"
```
修复

> 由于这里给了最高admin的权限，不建议在生产环境使用

```yaml
# NOTE: The service account `default:default` already exists in k8s cluster.
# You can create a new account following like this:
#---
#apiVersion: v1
#kind: ServiceAccount
#metadata:
#  name: <new-account-name>
#  namespace: <namespace>

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: fabric8-rbac
subjects:
  - kind: ServiceAccount
    # Reference to upper's `metadata.name`
    name: default
    # Reference to upper's `metadata.namespace`
    namespace: default
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```
验证如下：
```shell
$ kubectl get pod
NAME                                  READY   STATUS    RESTARTS      AGE
nginx-deploy-5ff9c9d8c7-grsfh         2/2     Running   0             110s
nginx-deploy-5ff9c9d8c7-hcbct         2/2     Running   0             111s

$ kubectl logs pod/nginx-deploy-5ff9c9d8c7-grsfh
Defaulted container "leader-elector" out of: leader-elector, nginx-deploy
nginx-deploy-5ff9c9d8c7-hcbct is the leader
I0303 14:39:10.661727       8 leaderelection.go:296] lock is held by nginx-deploy-5ff9c9d8c7-hcbct and has not yet expired
```
可以看到当前的主是nginx-deploy-5ff9c9d8c7-hcbct
接下来，我们将主pod杀掉，看是否会切换主：
```shell
$ kubectl delete pod/nginx-deploy-5ff9c9d8c7-hcbct
pod "nginx-deploy-5ff9c9d8c7-hcbct" deleted

$ kubectl logs pod/nginx-deploy-5ff9c9d8c7-jpqb8
I0303 14:48:04.285958       9 leaderelection.go:296] lock is held by nginx-deploy-5ff9c9d8c7-hcbct and has not yet expired
E0303 14:48:08.339644       9 event.go:257] Could not construct reference to: '&api.Endpoints{TypeMeta:unversioned.TypeMeta{Kind:"", APIVersion:""}, ObjectMeta:api.ObjectMeta{Name:"elect-nginx-deploy", GenerateName:"", Namespace:"default", SelfLink:"", UID:"34bd01ff-be9e-402b-bc46-39c1be5720ba", ResourceVersion:"1843444", Generation:0, CreationTimestamp:unversioned.Time{Time:time.Time{sec:63845073549, nsec:0, loc:(*time.Location)(0x240f0a0)}}, DeletionTimestamp:(*unversioned.Time)(nil), DeletionGracePeriodSeconds:(*int64)(nil), Labels:map[string]string(nil), Annotations:map[string]string{"control-plane.alpha.kubernetes.io/leader":"{\"holderIdentity\":\"nginx-deploy-5ff9c9d8c7-hcbct\",\"leaseDurationSeconds\":10,\"acquireTime\":\"2024-03-03T14:39:09Z\",\"renewTime\":\"2024-03-03T14:47:23Z\",\"leaderTransitions\":0}"}, OwnerReferences:[]api.OwnerReference(nil), Finalizers:[]string(nil)}, Subsets:[]api.EndpointSubset(nil)}' due to: 'selfLink was empty, can't make reference'. Will not report event: 'Normal' '%v became leader' 'nginx-deploy-5ff9c9d8c7-jpqb8'
nginx-deploy-5ff9c9d8c7-jpqb8 is the leader
I0303 14:48:08.339800       9 leaderelection.go:215] sucessfully acquired lease default/elect-nginx-deploy
```