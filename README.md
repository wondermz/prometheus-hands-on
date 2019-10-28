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
$ curl https://raw.githubusercontent.com/helm/helm/master/scripts/get > get_helm.sh
$ chmod 700 get_helm.sh
$ ./get_helm.sh

```

설치가 완료된 후 helm 을 초기화하고 tiller를 설치해야 합니다.
그 전에 tiller service account 를 생성하고 Cluster-admin role을 부여합니다.

```
$ kubectl -n kube-system create sa tiller
$ kubectl create clusterrolebinding tiller --clusterrole cluster-admin --serviceaccount=kube-system:tiller
```

helm을 설치하고 Service account를 생성했으니 다음의 명령어로 Tiller 를 설치하고, helm 의 stable chat 목록을 불러옵니다.

```

$ helm init --service-account tiller
$ helm repo update

```


### 1-2. Prometheus Operator 설치하기

설치가 완료된 후 helm chart 에서 Prometheus 와 Grafana Dashboard 를 한번에 설치하도록 도와주는 Prometheus Operator 를 사용하여 Prometheus를 설치합니다. 우선 Helm repo 에 Prometheus Operator 가 있는지 확인합니다.

```
$ helm search prometheus-operator

```

helm 은 설치 시 custom value 를 통해 간단하게 개인 환경에 맞는 설정으로 변경할 수 있습니다.
우선 특정 namespace 를 하나 생성하여 prometheus operator 를 설치합니다.
```
$ kubectl create namespace prometheus-operator
$ kcd prometheus-operator
# 여기서 --name은 helm install 시 생기는 release 이름을 지정한 것이며, --namespace는 특정 namespace 에 설치, 
# --set 부분은 custom value 를 사용하는 것입니다. yaml file 로 생성하여 지정해도 됩니다. 
$ helm install stable/prometheus-operator --name wondermz --namespace prometheus-operator --set grafana.adminPassword="wondermz"

```










