# AKS 구축하기

- 이전에 AWS EKS를 구축하던 것과 동일하게 CLI를 통해 클러스터를 구축한다.

- 마스터 노드 역할을 할 VM은 CentOS 8.2를 사용한다.

- 우선 구축과 배포 테스트를 하고 그 다음 네트워크 영역에 대한 설정을 동반한 클러스터 구성을 마친다.

- 그 다음은 팀에서 정한 이미지를 바탕으로 2개의 노드에 각각 파드 하나씩을 배포하고 두 파드를 묶어 로드밸런서 서비스로 배포한다.

## AZ CLI 설치

- `sudo rpm --import https://packages.microsoft.com/keys/microsoft.asc`

- 위 부분의 경우 rpm 에러가 발생하면 스킵하고 아래 두 명령어를 실행하고 설치 과정에서 위 부분을 수행할 지 묻는 부분에서 yes만 하면 된다.

- `sudo dnf install -y https://packages.microsoft.com/config/rhel/8/packages-microsoft-prod.rpm`

- `sudo dnf install azure-cli`

- Azure CLI도 파이썬으로 만들어져 있고 파이선 3.9버전으로 보인다. 2024년 2월 6일 기준 아 ㅋㅋ 생일인데 아 ㅋㅋ

## AKS 클러스터 생성

- `az login`

- Azure CLI 로그인 인증 방식은 웹 URL을 주고 그 웹에 접속해서 코드를 입력해서 인증하는 방식이다. 이 부분은 어떻게 자동화를 할 수 있을까?

- 생각보다 쉽게 할 수 있었다. `az login --identity`를 하면 그 VM이 생성된 계정 정보대로 로그인이 된다. AWS configure의 자동화보다 쉬운 것 같다.

- 클러스터를 생성한 다음 그 클러스터에 대한 권한을 전달 받는 방식인 것 같다.

- `az aks create` -> `az aks install-cli` -> `az aks get-credentials` 후 이제 kubectl 명령어를 사용하면 AKS 클러스터가 자동으로 노드 VM을 생성하고 서비스 로드밸런서를 생성해줄 것으로 보인다.

- `az aks create --resource-group myResourceGroup --name myAKSCluster --enable-managed-identity --node-count 1 --generate-ssh-keys`

- `--node-count n` 3이 default, `az aks scale`로 이후 변경 가능

- `--kubernetes-version [버전]` 사용할 쿠버네티스 버전 명시? `az aks install-cli`로 버전을 정해서 kubectl을 받은 다음 그 버전에 맞게 aks 클러스터 생성하는 듯

- `--vnet-subnet-id [서브넷의 id]` - 클러스터를 배포할(worker node)들이 위치할 서브넷의 지정한다.

- `az network vnet list --resource-group semi-2-weus3 `로 서브넷의 ID를 알 수 있다. `"id": "/subscriptions/c339a2ea-3583-481d-87b6-33490df12eb2/resourceGroups/semi-2-weus3/providers/Microsoft.Network/virtualNetworks/vn-weus3/subnets/pvt-subnet"` 중 `c339a2ea-3583-481d-87b6-33490df12eb2`?

- `--tier` 노드로 생성할 VM의 인스턴스 타입을 지정한다?

- `az aks get-credentials --resource-group semi-2-seoul --name myAKSCluster`

- 이후 kubectl로 nginx 이미지를 pull해서 pod를 생성한 다음 그 pod를 LoadBalancer 서비스로 expose하면 퍼블릭 IP가 부여된 상태로 배포되고 접속도 가능하다.

- `--enable-cluster-autoscaler --min-count 1 --max-count 3`

- `az aks update --resource-group [리소스 그룹 이름] --name [클러스터 이름] --enable-cluster-autoscaler --min-count 1 --max-count 3`

- 오토 스케일링 옵션 CPU 코어 제한 고려하면 일단 노드 그룹은 지금 하나밖에 안되지만 일단 오토 스케일링을 적용한다.

- `az aks update --resource-group myResourceGroup --name myAKSCluster --enable-cluster-autoscaler --min-count 1 --max-count 3` 다 생성한 후에도 업데이트 명령을 통해 오토 스케일링으로 변경이 가능하다.

- `az aks update --resource-group myResourceGroup --name myAKSCluster --disable-cluster-autoscaler`로 취소할 수도 있다.

- https://learn.microsoft.com/en-us/cli/azure/aks?view=azure-cli-latest - AKS 클러스터 생성

- https://learn.microsoft.com/ko-kr/azure/aks/cluster-autoscaler?tabs=azure-cli AKS 클러스터에 오토 스케일링 적용, 오토 스케일링 세부 설정

- `az network route-table list [리소스 그룹 이름]`

## Route 53에 Azure AKS LB IP 도메인으로 등록

- `az aks create --resource-group semi-2-weus3 --name AKSCluster --enable-managed-identity --vnet-subnet-id /subscriptions/c339a2ea-3583-481d-87b6-33490df12eb2/resourceGroups/semi-2-weus3/providers/Microsoft.Network/virtualNetworks/vn-weus3/subnets/pvt-subnet --generate-ssh-keys --node-count 1`

- 로 미리 생성해둔 프라이빗 서브넷에 AKS 클러스터 생성

- `sudo az aks install-cli --client-version 1.27.7`

- `az aks get-credentials --resource-group semi-2-weus3 --name AKSCluster`

- `kubectl get nodes -o wide`

- 노드 풀로 ssh

- 노드 풀 안에서 `apt update` 수행

- 미리 구축한 NAT와 연결된 프라이빗 서브넷에 클러스터가 생성되고 그 NAT를 통해 인터넷 통신 가능한 것을 확인

- 결국 AKS 노드 풀은 따로 생성된다. EKS와 마찬가지로

- 내가 미리 구축해둔 서브넷에서 완전히 통제할 수는 없다. 클러스터와 내 프라이빗 서브넷이 라우팅 테이블로 연결되어 있을 뿐

- 공용 IP가 부족하니 프라이빗 서브넷, NAT, NAT IP 모두 삭제하고 Azure VM에서 AKS 클러스터를 제어하는 것을 목적으로 하자.

- `az aks create --resource-group semi-2-weus3 --name AKSCluster --enable-managed-identity --generate-ssh-keys --node-count 1`

- 절대 의미없는 것이 아니었다... 다행이네...

- 노드 풀로 SSH가 가능했던 부분 등 위에서 시도했던 클러스터 구축은 미리 구성한 확실히 프라이빗 서브넷에 위치한 것과 같이 동작했다. 공용 IP 이슈가 없었다면 온전히 동작하는지도 확인이 가능했을 것이다.

- 프라이빗으로 만들지 않고 일반 서브넷으로 만든 다음에 (NAT 없이) 그 서브넷에 클러스터 구축하고 로드밸런서로 뺄 수 있다면 이것도 하나의 증명일 듯 하다.

- wordpress:4.8.2

## 프로젝트에서 사용할 Wordpress 이미지 빌드해서 Azure 레지스트리에 등록

- 우선은 Azure 레지스트리에 등록하는 방법, 그리고 그 레지스트리에서 이미지를 불러와 배포하는 것을 먼저 해본다.

- 프로젝트에서는 사설 레지스트리를 사용하는데 일단은 직접 빌드한 이미지를 바탕으로 쿠버네티스 환경에 배포하는 것이 목적이기 때문에 클라우드 레지스트리를 사용한다.

- https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-init/

## 로컬에서 Azure CLI 설치, 로그인

- WSL2 우분투에서 Azure CLI를 사용해서 쿠버네티스 클러스터를 구축한다.

- CPU 할당량 문제로 노드를 2개 사용하기 위해서 마스터 노드를 사용하지 않는 것으로 결정

## semi2

```bash
az aks create --resource-group semi-2-weus3 --name AKSCluster --enable-managed-identity --vnet-subnet-id /subscriptions/c339a2ea-3583-481d-87b6-33490df12eb2/resourceGroups/semi-2-weus3/providers/Microsoft.Network/virtualNetworks/vn-weus3/subnets/pvt-subnet --generate-ssh-keys --node-count 2 --network-plugin azure --network-plugin-mode overlay --network-policy calico  --pod-cidr 10.244.0.0/16
```

```bash
sudo az aks install-cli --client-version 1.27.7
```

```bash
az aks get-credentials --resource-group semi-2-weus3 --name AKSCluster
```

```bash
kubectl create secret generic mysql-pass --from-literal=password="wppass"
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress-mysql
  labels:
    app: wordpress
spec:
  selector:
    matchLabels:
      app: wordpress
      tier: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: wordpress
        tier: mysql
    spec:
      containers:
        - image: mysql:5.6
          name: mysql
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-pass
                  key: password
          ports:
            - containerPort: 3306
              name: mysql
---
apiVersion: v1
kind: Service
metadata:
  name: wordpress-mysql
  labels:
    app: wordpress
spec:
  type: ClusterIP
  ports:
    - port: 3306
  selector:
    app: wordpress
    tier: mysql
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
  labels:
    app: wordpress
spec:
  replicas: 2
  selector:
    matchLabels:
      app: wordpress
      tier: frontend
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: wordpress
        tier: frontend
    spec:
      containers:
        - image: wordpress:4.8.2
          name: wordpress
          env:
            - name: WORDPRESS_DB_HOST
              value: wordpress-mysql
            - name: WORDPRESS_DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-pass
                  key: password
          ports:
            - containerPort: 80
              name: wordpress
---
apiVersion: v1
kind: Service
metadata:
  name: wordpress
  labels:
    app: wordpress
spec:
  ports:
    - port: 80
  selector:
    app: wordpress
    tier: frontend
  type: LoadBalancer
```

```bash
kubectl apply -f wordpress-mysql.yaml
```
