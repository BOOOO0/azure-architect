# AKS 구축하기

- 이전에 AWS EKS를 구축하던 것과 동일하게 CLI를 통해 클러스터를 구축한다.

- 마스터 노드 역할을 할 VM은 CentOS 8.2를 사용한다.

- 우선 구축과 배포 테스트를 하고 그 다음 네트워크 영역에 대한 설정을 동반한 클러스터 구성을 마친다.

- 그 다음은 팀에서 정한 이미지를 바탕으로 2개의 노드에 각각 파드 하나씩을 배포하고 두 파드를 묶어 로드밸런서 서비스로 배포한다.

## AZ CLI 설치

- `sudo rpm --import https://packages.microsoft.com/keys/microsoft.asc`

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

- `--tier` 노드로 생성할 VM의 인스턴스 타입을 지정한다?

- `az aks get-credentials --resource-group semi-2-seoul --name myAKSCluster`

- 이후 kubectl로 nginx 이미지를 pull해서 pod를 생성한 다음 그 pod를 LoadBalancer 서비스로 expose하면 퍼블릭 IP가 부여된 상태로 배포되고 접속도 가능하다.

- `--enable-cluster-autoscaler --min-count 1 --max-count 3`

- `az aks -r [리소스 그룹 이름] -n [클러스터 이름] update --enable-cluster-autoscaler --min-count 1 --max-count 3`

- 오토 스케일링 옵션 CPU 코어 제한 고려하면 일단 노드 그룹은 지금 하나밖에 안되지만 일단 오토 스케일링을 적용한다.

- `az aks update --resource-group myResourceGroup --name myAKSCluster --enable-cluster-autoscaler --min-count 1 --max-count 3` 다 생성한 후에도 업데이트 명령을 통해 오토 스케일링으로 변경이 가능하다.

- `az aks update --resource-group myResourceGroup --name myAKSCluster --disable-cluster-autoscaler`로 취소할 수도 있다.

- https://learn.microsoft.com/en-us/cli/azure/aks?view=azure-cli-latest

- https://learn.microsoft.com/ko-kr/azure/aks/cluster-autoscaler?tabs=azure-cli

## 프로젝트에서 사용할 Wordpress 이미지 빌드해서 Azure 레지스트리에 등록

- 우선은 Azure 레지스트리에 등록하는 방법, 그리고 그 레지스트리에서 이미지를 불러와 배포하는 것을 먼저 해본다.

- 프로젝트에서는 사설 레지스트리를 사용하는데 일단은 직접 빌드한 이미지를 바탕으로 쿠버네티스 환경에 배포하는 것이 목적이기 때문에 클라우드 레지스트리를 사용한다.
