# prometheus-hands-on

이 HandsOn은 EKS-hands-on 실습에 이어서 진행되는 handson입니다.

Prometheus 를 Kubernetes 클러스터 내에 설치하는 방법은 여러가지가 있지만, 는

helm chart 와 Prometheus Operator 를 사용하여 간단하게 설치하고, 실제 운용환경에서 손쉽게 운용할 수 있습니다.

Helm Chart 와 Prometheus Operator 에 대한 자세한 아키텍쳐와 동작 구조은 해당 문서를 참조하시기 바랍니다.

* [Helm Chart](https://helm.sh/)
* [Prometheus Operator](https://coreos.com/blog/the-prometheus-operator.html)


## 1. 기본 환경 만들기

이전에 EKS HandsOn 수업을 따라 오신 분들은 kubectl 전용 인스턴스에서 시작하시면 됩니다.

Kubernetes Cluster 가 준비되지 않으신 분들은 [EKS HandsOn](https://github.com/wondermz/eks-hands-on)을 따라하신 후 이 HandsOn을 따라하시면 됩니다.

### 1-1. Helm 설치하기

* Kubectl EC2 접속 후 

kubectl ec2 인스턴스에 접속한 후 다음의 명령어를 입력하여 설치 스크립트를 받고 실행하시면 됩니다.

```
$ curl https://raw.githubusercontent.com/kubernetes/helm/master/scripts/get > get_helm.sh

$ chmod +x get_helm.sh

$ ./get_helm.sh

```

### 1-2. Tiller 설정 및 Tiler Service Account 생

Helm 은 Tiller 라는 deploy Service 를 통해 kubernetes resource 들을 설치하기 때문에,
Tiller 가 사용할 서비스 계정을 만들어야 합니다. 그 이후 helm 초기화를 통해 Prometheus 설치를 위한 helm 기본 준비를 합니다.

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

이제 config 파일을 적용한 후, tiller service account를 사용하여 helm init 을 적용하시면 됩니다. 

```

$ kubectl apply -f ~/rbac.yaml
$ helm init --service-account tiller

```


## 2. Prometheus Operator 설치하기


설치가 완료된 후 helm chart 에서 Prometheus 와 Grafana Dashboard 를 한번에 설치하도록 도와주는 Prometheus Operator 를 사용하여 Prometheus를 설치합니다. 우선 Helm repo 에 Prometheus Operator 가 있는지 확인합니다.


```
$ helm repo update
$ helm search prometheus-operator
```

~~~
NAME                      	CHART VERSION	APP VERSION	DESCRIPTION
stable/prometheus-operator	6.21.0       	0.32.0     	Provides easy monitoring definitions for Kubernetes servi...
~~~
우선 특정 namespace 를 하나 생성하여 prometheus operator 를 설치합니다.



```
$ kubectl create namespace prometheus-operator
$ kcd prometheus-operator
```

***

### 2-1. Helm Custom Value 설정하기  

helm 은 설치 시 custom value 를 통해 간단하게 개인 환경에 맞는 설정으로 변경할 수 있습니다.

Grafana Dashboard 접속과 password 설정을 위해 prometheus value yaml file 을 하나 생성합니다.  

혹시 Prometheus Operator에 추가로 설정하실 value 가 있다면 , 관련 value 설정은 [github](https://github.com/helm/charts/tree/master/stable/prometheus-operator)를 참조하시면 됩니다.

~~~
vi values.yml
~~~

~~~
#values.yml

grafana: 
  adminPassword: wondermz
  service: 
    type: LoadBalancer
~~~
***
### 2-2. Helm Chart 로 Prometheus Operator 설치하기 




기본적으로 Prometheus Operator stable version chart 를 사용하며, --name 은 helm chart 의 release name, --namespace 는 namespace 지정, -f 는 해당 파일 지정 tag 를 사용하여 설치하시면 됩니다. 

~~~
$ helm install stable/prometheus-operator --name wondermz --namespace prometheus-operator -f values.yml 
~~~

약 1분정도 기다리시면 helm chart 에 있는 모든 service , Pod 등이 생성됩니다.

~~~

$ k get all


NAME                                                         READY   STATUS    RESTARTS   AGE
pod/alertmanager-wondermz-prometheus-operat-alertmanager-0   2/2     Running   0          17m
pod/prometheus-wondermz-prometheus-operat-prometheus-0       3/3     Running   1          17m
pod/wondermz-grafana-68cf89564b-5749l                        2/2     Running   0          17m
pod/wondermz-kube-state-metrics-856dcdcc67-h6ncg             1/1     Running   0          17m
pod/wondermz-prometheus-node-exporter-7rh2z                  1/1     Running   0          17m
pod/wondermz-prometheus-node-exporter-chtfd                  1/1     Running   0          17m
pod/wondermz-prometheus-node-exporter-fjsm6                  1/1     Running   0          17m
pod/wondermz-prometheus-node-exporter-pg8d4                  1/1     Running   0          17m
pod/wondermz-prometheus-node-exporter-tctvq                  1/1     Running   0          17m
pod/wondermz-prometheus-operat-operator-54545684f5-9tpfc     2/2     Running   0          17m

NAME                                              TYPE           CLUSTER-IP       EXTERNAL-IP                                                                    PORT(S)                      AGE
service/alertmanager-operated                     ClusterIP      None             <none>                                                                         9093/TCP,9094/TCP,9094/UDP   17m
service/prometheus-operated                       ClusterIP      None             <none>                                                                         9090/TCP                     17m
service/wondermz-grafana                          LoadBalancer   10.100.247.37    a46547715f95a11e99a4a02a530d5340-1940580701.ap-northeast-2.elb.amazonaws.com   80:32046/TCP                 17m
service/wondermz-kube-state-metrics               ClusterIP      10.100.68.62     <none>                                                                         8080/TCP                     17m
service/wondermz-prometheus-node-exporter         ClusterIP      10.100.49.97     <none>                                                                         9100/TCP                     17m
service/wondermz-prometheus-operat-alertmanager   ClusterIP      10.100.231.204   <none>                                                                         9093/TCP                     17m
service/wondermz-prometheus-operat-operator       ClusterIP      10.100.127.142   <none>                                                                         8080/TCP,443/TCP             17m
service/wondermz-prometheus-operat-prometheus     ClusterIP      10.100.233.125   <none>                                                                         9090/TCP                     17m

NAME                                               DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
daemonset.apps/wondermz-prometheus-node-exporter   5         5         5       5            5           <none>          17m

NAME                                                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/wondermz-grafana                      1/1     1            1           17m
deployment.apps/wondermz-kube-state-metrics           1/1     1            1           17m
deployment.apps/wondermz-prometheus-operat-operator   1/1     1            1           17m

NAME                                                             DESIRED   CURRENT   READY   AGE
replicaset.apps/wondermz-grafana-68cf89564b                      1         1         1       17m
replicaset.apps/wondermz-kube-state-metrics-856dcdcc67           1         1         1       17m
replicaset.apps/wondermz-prometheus-operat-operator-54545684f5   1         1         1       17m

NAME                                                                    READY   AGE
statefulset.apps/alertmanager-wondermz-prometheus-operat-alertmanager   1/1     17m
statefulset.apps/prometheus-wondermz-prometheus-operat-prometheus       1/1     17m
~~~

***

### 2-3. Grafana Dashboard 접속하기


이렇게 모든 관련 service 들이 running 상태에 들어간 후, 실제 Grafana DashBoard 에 접속하기 위해 AWS ELB service 주소로 접속합니다.

명령어로 친 후 Extermal-ip 의 주소를 주소창에 입력해서 접속하시면 됩니다.

만약 Externel-ip 생성이 Pending 상태일 때, 약 5-10분정도 기다리시면 생성됩니다.


~~~

$k get svc wondermz-grafana -o wide

NAME               TYPE           CLUSTER-IP      EXTERNAL-IP                                                                    PORT(S)        AGE   SELECTOR
wondermz-grafana   LoadBalancer   10.100.247.37   a46547715f95a11e99a4a02a530d5340-1940580701.ap-northeast-2.elb.amazonaws.com   80:32046/TCP   15m   app=grafana,release=wondermz

~~~


접속 id 는 admin, 비밀번호는 wondermz 로 grafana dashboard 에 접속하시면 default dashboard 가 있는 것을 확인할 수 있습니다.










