root@ip-172-31-30-83:/home/ubuntu/sa# kubectl delete pod test-pod; kubectl delete -f saRoleBinding.yaml; kubectl delete -f Role.yaml; kubectl delete sa newsa


root@ip-172-31-30-83:/home/ubuntu/sa# kubectl get sa
NAME      SECRETS   AGE
default   1         190d
mysa      1         3d16h
root@ip-172-31-30-83:/home/ubuntu/sa# kubectl get secret
NAME                  TYPE                                  DATA   AGE
default-token-rspfg   kubernetes.io/service-account-token   3      190d
mysa-token-m2vvp      kubernetes.io/service-account-token   3      3d16h
root@ip-172-31-30-83:/home/ubuntu/sa# kubectl create sa newsa
serviceaccount/newsa created
root@ip-172-31-30-83:/home/ubuntu/sa# kubectl get secret
NAME                  TYPE                                  DATA   AGE
newsa-token-npxgk     kubernetes.io/service-account-token   3      27s

root@ip-172-31-30-83:/home/ubuntu/sa# cat pod-sa.yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: test-pod
  name: test-pod
spec:
  serviceAccount: newsa
  containers:
  - image: nginx
    name: test-pod
root@ip-172-31-30-83:/home/ubuntu/sa# kubectl apply -f pod-sa.yaml
pod/test-pod created

root@ip-172-31-30-83:/home/ubuntu/sa# kubectl describe pod test-pod

root@ip-172-31-30-83:/home/ubuntu/sa# kubectl get service
NAME         TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1     <none>        443/TCP   182d

root@ip-172-31-30-83:/home/ubuntu/sa# kubectl exec -it test-pod -- bash
root@test-pod:/# ls -l /var/run/secrets/kubernetes.io/serviceaccount
total 0
lrwxrwxrwx 1 root root 13 Mar 19 22:28 ca.crt -> ..data/ca.crt
lrwxrwxrwx 1 root root 16 Mar 19 22:28 namespace -> ..data/namespace
lrwxrwxrwx 1 root root 12 Mar 19 22:28 token -> ..data/token

root@test-pod:/# TOKEN=`cat /var/run/secrets/kubernetes.io/serviceaccount/token`
root@test-pod:/# curl https://kubernetes -k --header "Authorization: Bearer $TOKEN"
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {

  },
  "status": "Failure",
  "message": "forbidden: User \"system:serviceaccount:default:newsa\" cannot get path \"/\"",
  "reason": "Forbidden",
  "details": {

  },
  "code": 403
}

root@ip-172-31-30-83:/home/ubuntu/sa# cat Role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-read-only
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
root@ip-172-31-30-83:/home/ubuntu/sa# kubectl get role
NAME     CREATED AT
pod-ro   2021-12-03T00:09:39Z

root@ip-172-31-30-83:/home/ubuntu/sa# kubectl apply -f Role.yaml
role.rbac.authorization.k8s.io/pod-read-only created

apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: bind-pod-read-only
  namespace: default
subjects:
  - kind: ServiceAccount
    namespace: default
    name: newsa
roleRef:
  kind: Role
  name: pod-read-only
  apiGroup: rbac.authorization.k8s.io



root@ip-172-31-30-83:/home/ubuntu/sa# kubectl apply -f saRoleBinding.yaml
rolebinding.rbac.authorization.k8s.io/bind-pod-read-only created

kubectl exec -it test-pod -- bash

echo "deb http://us.archive.ubuntu.com/ubuntu vivid main universe" >> /etc/apt/sources.list
apt-get update
apt-get install jq

root@test-pod:/# TOKEN=`cat /var/run/secrets/kubernetes.io/serviceaccount/token`
root@test-pod:/# curl https://kubernetes/api/v1/namespaces/default/pods -k --header "Authorization: Bearer $TOKEN"  | jq '.items[].metadata.name'

"some-deployment-6dfcc5b79f-cvjtc"
"some-deployment-6dfcc5b79f-hvbqp"
"test-pod"
