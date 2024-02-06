# Google Cloud Platform

- gcp메인 화면에 있는 gcp에서 제공하는 시나리오를 보면 terraform 코드 링크가 있다.

- gcp도 여러 시나리오에 대한 테라폼 코드를 깃허브에서 관리하고 제공한다. 모듈로 호출 가능한 듯 하다.

# VPC

- 리전별로 구분되는 VPC가 아니라 글로벌로 VPC가 만들어지고 그 안에 서브넷을 여러 리전에 생성할 수 있다.

- GCP도 Azure처럼 서브넷 단위에서 가용 영역을 나누지 않는다.

- public과 private의 차이도 Azure 처럼 Public IP를 부여했는지 아닌지의 차이이다.

- asia-northeast-3이 서울리전이다. 1 - 도쿄, 2 - 오사카

- 프로그램이 코드상에 IP가 그때그때 만드는대로 하드코딩되어서 클라우드 마이그레이션이 어려웠던 사례 - IP 대역 잘 나누고 문서 작성 잘하라는 얘기

- 서브넷에 보안 그룹과 같은 방화벽 규칙을 적용할 수 있다.

# Cloud DNS

-

# Compute Engine

- EC2, Azure VM같은 컴퓨팅 리소스이다.
