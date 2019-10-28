# prometheus-hands-on

이 HandsOn은 EKS-hands-on 실습에 이어서 진행되는 handson입니다.

Prometheus 를 Kubernetes 클러스터 내에 설치하는 방법은 여러가지가 있지만, helm chart 와 Prometheus Operator 를 사용하여 간단하게 설치하고, 실제 운용환경에서 손쉽게 운용할 수 있습니다.

Helm Chart 와 Prometheus Operator 에 대한 자세한 내용은 해당 문서를 참조하시기 바랍니다.

* [Helm Chart](https://helm.sh/)
* [Prometheus Operator](https://coreos.com/blog/the-prometheus-operator.html)


## 1. 기본 환경 만들기

이전에 EKS HandsOn 수업을 따라 오신 분들은 kubectl 전용 인스턴스에서 시작하시면 됩니다.
Kubernetes Cluster 가 준비되지 않으신 분들은 [EKS HandsOn](https://github.com/wondermz/eks-hands-on)을 따라하신 후 이 HandsOn을 따라하시면 됩니다.

### 1-1. Helm 설치하기

**Kubectl EC2 접속 후 **

kubectl ec2 인스턴스에 접속한 후 다음의 명령어를 입력하여 설치 스크립트를 받고 실행하시면 됩니다.

```
$ curl https://raw.githubusercontent.com/kubernetes/helm/master/scripts/get > get_helm.sh

$ chmod +x get_helm.sh

$ ./get_helm.sh

```

### 1-2. Tiller 설정 및 Service Account 생

Helm 은 kubernetes 클러스터에 대한 Tiller 설치에 대해 의존함으로 Tiller 가 사용할 서비스 계정을 만들어야 합니다. 그 이후 클러스터에 적용합니다.
```
cat <<EoF > ~/rbac.yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tiller
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: tiller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: tiller
    namespace: kube-system
EoF

```

이후에 config 파일을 적용한 후, tiller service account를 사용하여 helm init 을 적용하시면 됩니다. 

```

$ kubectl apply -f ~/rbac.yaml
$ helm init --service-account tiller

```


### 1-2. Prometheus Operator 설치하기

설치가 완료된 후 helm chart 에서 Prometheus 와 Grafana Dashboard 를 한번에 설치하도록 도와주는 Prometheus Operator 를 사용하여 Prometheus를 설치합니다. 우선 Helm repo 에 Prometheus Operator 가 있는지 확인합니다.

```
$ helm repo update
$ helm search prometheus-operator

```

helm 은 설치 시 custom value 를 통해 간단하게 개인 환경에 맞는 설정으로 변경할 수 있습니다.
우선 특정 namespace 를 하나 생성하여 prometheus operator 를 설치합니다.

```
$ kubectl create namespace prometheus-operator
$ kcd prometheus-operator

## 여기서 --name은 helm install 시 생기는 release 이름을 지정한 것이며, --namespace는 특정 namespace 에 설치, 
## --set 부분은 custom value 를 사용하는 것입니다. yaml file 로 생성하여 지정해도 됩니다. 
##지금은 간단하게 설치하기 위해 이렇게 직접 명령어로 변수 지정하겠습니다.

$ helm install stable/prometheus-operator --name wondermz --namespace prometheus-operator --set grafana.adminPassword="wondermz" --set grafana.service.type=LoadBalancer 
```

약 1분정도 기다리시면 helm chart 에 있는 모든 service , Pod 등이 생성됩니다.

~~~

$ k get all


NAME                                                         READY   STATUS              RESTARTS   AGE
pod/alertmanager-wondermz-prometheus-operat-alertmanager-0   2/2     Running             0          33s
pod/prometheus-wondermz-prometheus-operat-prometheus-0       0/3     ContainerCreating   0          23s
pod/wondermz-grafana-68cf89564b-h6cjj                        2/2     Running             0          44s
pod/wondermz-kube-state-metrics-856dcdcc67-q5l2q             1/1     Running             0          44s
pod/wondermz-prometheus-node-exporter-2rb57                  1/1     Running             0          44s
pod/wondermz-prometheus-node-exporter-5ck8n                  1/1     Running             0          44s
pod/wondermz-prometheus-node-exporter-8sfjb                  1/1     Running             0          44s
pod/wondermz-prometheus-node-exporter-kq896                  1/1     Running             0          44s
pod/wondermz-prometheus-node-exporter-t9cz8                  1/1     Running             0          44s
pod/wondermz-prometheus-operat-operator-54545684f5-54777     2/2     Running             0          44s

NAME                                              TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
service/alertmanager-operated                     ClusterIP      None             <none>        9093/TCP,9094/TCP,9094/UDP   33s
service/prometheus-operated                       ClusterIP      None             <none>        9090/TCP                     23s
service/wondermz-grafana                          LoadBalancer   10.100.120.84    <pending>     80:30296/TCP                 45s
service/wondermz-kube-state-metrics               ClusterIP      10.100.7.201     <none>        8080/TCP                     45s
service/wondermz-prometheus-node-exporter         ClusterIP      10.100.148.159   <none>        9100/TCP                     45s
service/wondermz-prometheus-operat-alertmanager   ClusterIP      10.100.159.106   <none>        9093/TCP                     45s
service/wondermz-prometheus-operat-operator       ClusterIP      10.100.163.234   <none>        8080/TCP,443/TCP             44s
service/wondermz-prometheus-operat-prometheus     ClusterIP      10.100.151.242   <none>        9090/TCP                     45s

NAME                                               DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
daemonset.apps/wondermz-prometheus-node-exporter   5         5         5       5            5           <none>          44s

NAME                                                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/wondermz-grafana                      1/1     1            1           44s
deployment.apps/wondermz-kube-state-metrics           1/1     1            1           44s
deployment.apps/wondermz-prometheus-operat-operator   1/1     1            1           44s

NAME                                                             DESIRED   CURRENT   READY   AGE
replicaset.apps/wondermz-grafana-68cf89564b                      1         1         1       44s
replicaset.apps/wondermz-kube-state-metrics-856dcdcc67           1         1         1       44s
replicaset.apps/wondermz-prometheus-operat-operator-54545684f5   1         1         1       44s

NAME                                                                    READY   AGE
statefulset.apps/alertmanager-wondermz-prometheus-operat-alertmanager   1/1     33s
statefulset.apps/prometheus-wondermz-prometheus-operat-prometheus       0/1     23s


~~~

이렇게 모든 관련 service 들이 running 상태에 들어간 후, 실제 Grafana DashBoard 에 접속하기 위해 AWS ELB service 주소로 접속합니다.
명령어로 친 후 Extermal-ip 의 주소를 주소창에 입력해서 접속하시면 됩니다.
~~~

$k get svc wondermz-grafana -o wide

NAME               TYPE           CLUSTER-IP      EXTERNAL-IP                                                                    PORT(S)        AGE   SELECTOR
wondermz-grafana   LoadBalancer   10.100.247.37   a46547715f95a11e99a4a02a530d5340-1940580701.ap-northeast-2.elb.amazonaws.com   80:32046/TCP   15m   app=grafana,release=wondermz

~~~






