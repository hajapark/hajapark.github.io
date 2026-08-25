---
title: "Rails 앱을 AWS에 배포하기 전 설계하기: EC2, RDS, Docker, Kamal"
date: 2026-08-23 23:59:00 +0900
categories: [DevOps, AWS]
tags: [Rails, AWS, VPC, EC2, RDS, PostgreSQL, Docker, Kamal, ReverseProxy, Deployment]
---

> **한 줄 요약**: 작은 Rails 서비스를 처음 배포할 때는 공개 EC2, 비공개 RDS, Docker, Kamal을 조합하면 각 구성요소의 역할을 직접 이해하면서 확장 가능한 출발점을 만들 수 있다.

Rails 앱을 로컬 컴퓨터 밖에서 실행하려면 단순히 소스 코드를 서버에 복사하는 것보다 많은 결정이 필요하다. 서버는 어디에 둘지, 데이터베이스는 어떻게 보호할지, 실행 환경은 어떻게 동일하게 만들지, 새 버전은 어떻게 교체할지를 정해야 한다.

이 글은 특정 프로젝트의 IP 주소나 인스턴스 이름을 따라 하는 절차가 아니다. 작은 Rails 서비스를 AWS에 처음 배포한다고 가정하고, 각 기술이 왜 필요한지부터 첫 배포 직전의 확인 과정까지 설명한다.

## 1. 먼저 전체 그림을 이해한다

처음부터 여러 서버와 로드 밸런서를 운영하면 비용과 학습 범위가 함께 커진다. 트래픽이 많지 않은 서비스라면 웹 서버 한 대와 관리형 데이터베이스 하나로 시작할 수 있다.

```text
인터넷
  │ HTTP/HTTPS
  ▼
Internet Gateway
  │
  ▼
Public subnet A ── EC2
                      │ PostgreSQL 5432
                      ▼
                 RDS PostgreSQL
                 Private subnet A
                 Private subnet B도 DB subnet group에 포함
```

여기서 등장하는 서비스의 역할은 다음과 같다.

- **EC2(Elastic Compute Cloud)**: AWS에서 빌리는 가상 컴퓨터다. Rails와 Docker가 이 서버에서 실행된다.
- **RDS(Relational Database Service)**: PostgreSQL 설치, 백업, 장애 복구 같은 운영 작업을 AWS가 일부 대신 관리해 주는 데이터베이스 서비스다.
- **VPC(Virtual Private Cloud)**: EC2와 RDS를 배치하는 AWS 내부의 전용 네트워크다.
- **Docker**: 애플리케이션과 실행에 필요한 라이브러리를 하나의 이미지로 포장한다.
- **Kamal**: Docker 이미지를 만들고 서버에 전달한 뒤 새 컨테이너로 교체하는 배포 도구다.
- **Kamal Proxy**: 외부 요청을 적절한 Rails 컨테이너로 전달하고 배포 중 트래픽 전환과 HTTPS를 담당하는 리버스 프록시다.

단일 EC2 구조는 완성형이 아니라 시작점이다. 서버 장애에도 서비스를 계속해야 하거나 트래픽이 증가하면 ALB(Application Load Balancer), 여러 EC2, 중앙 이미지 레지스트리, S3 같은 구성요소를 추가한다.

## 2. AWS 계정과 비용부터 보호한다

### MFA와 IAM이란?

AWS root 계정은 결제와 계정 삭제를 포함한 모든 권한을 가진다. 일상적인 배포 작업에 계속 사용하기에는 권한이 너무 크다.

- **IAM(Identity and Access Management)**은 사용자와 역할별로 AWS 권한을 나누는 서비스다.
- **MFA(Multi-Factor Authentication)**는 비밀번호 외에 인증 앱 등의 추가 인증 수단을 요구한다.

리소스를 만들기 전에 다음을 준비한다.

- root 계정과 일상용 IAM 사용자에 MFA 적용
- 평소 작업에는 필요한 권한만 가진 IAM 사용자 또는 역할 사용
- AWS Budgets에서 월 예산과 알림 기준 설정
- 실습 종료 후 중지할 것과 삭제할 것을 미리 기록

### 작은 구성에서도 비용이 생기는 곳

EC2와 RDS를 중지하더라도 EBS 스토리지, RDS 백업, 스냅샷 같은 비용은 남을 수 있다. AWS는 자동 할당 주소와 Elastic IP를 포함한 public IPv4 주소에도 요금을 부과한다.

NAT Gateway, Load Balancer, Multi-AZ RDS도 편리하지만 지속 비용과 운영 범위를 늘린다. 이 구성에서는 다음 이유로 제외했다.

- 웹 서버 한 대만 사용하므로 ALB가 아직 필요하지 않다.
- 인터넷 통신이 필요한 EC2를 공개 서브넷에 두므로 NAT Gateway가 필요하지 않다.
- 학습 단계에서는 RDS를 Single-AZ로 시작하되 백업과 복구 방법은 별도로 준비한다.

비용 절감을 위해 기능을 빼는 경우에는 그 기능이 해결하던 위험도 함께 기록해야 한다. 예를 들어 Multi-AZ를 빼면 장애 조치 능력이 줄어든다.

## 3. VPC, 서브넷, 라우팅

### VPC와 서브넷의 관계

VPC가 프로젝트 전용 네트워크라면 **서브넷(subnet)**은 그 네트워크를 용도별로 나눈 구역이다.

- **공개 서브넷**: Internet Gateway로 향하는 경로가 있다. 공인 IP도 가진 EC2는 인터넷과 통신할 수 있다.
- **비공개 서브넷**: 인터넷에서 직접 들어오는 경로가 없다. RDS처럼 외부에 공개할 필요가 없는 리소스를 둔다.
- **라우팅 테이블**: 네트워크 요청을 어디로 보낼지 정하는 표다.
- **Internet Gateway**: VPC와 인터넷을 연결하는 관문이다.

공개 서브넷에 있다는 사실만으로 서버가 공개되는 것은 아니다. EC2에 공인 주소가 있어야 하고, 라우팅과 보안 그룹도 함께 맞아야 한다.

### RDS 서브넷은 왜 두 개 이상 필요한가?

RDS의 **DB subnet group**은 데이터베이스를 배치할 수 있는 서브넷 목록이다. 일반적인 AWS 리전에서는 서로 다른 가용 영역(Availability Zone)의 서브넷을 둘 이상 포함해야 한다.

두 서브넷을 넣었다고 자동으로 Multi-AZ가 되는 것은 아니다. Single-AZ와 Multi-AZ는 데이터베이스 생성 시 별도로 선택한다. 두 번째 서브넷은 장애 복구와 향후 Multi-AZ 전환을 위한 배치 선택지를 제공한다.

### NAT Gateway는 언제 필요한가?

NAT Gateway는 비공개 서브넷의 서버가 인터넷으로 나가되 인터넷에서 직접 접속받지는 않도록 돕는다. “서버가 외부 API를 사용한다”는 이유만으로 항상 필요한 것은 아니다.

```text
공개 서브넷의 EC2 → Internet Gateway로 직접 나감
비공개 서브넷의 EC2 → 외부 통신이 필요하면 NAT Gateway 검토
비공개 RDS         → 보통 외부 인터넷 통신이 필요하지 않음
```

## 4. 보안 그룹으로 통신 범위를 제한한다

**보안 그룹(Security Group)**은 EC2와 RDS 앞에서 동작하는 상태 기반 가상 방화벽이다.

- **인바운드**: 외부에서 리소스로 들어오는 요청
- **아웃바운드**: 리소스가 외부로 보내는 요청
- **출발지(source)**: 요청이 시작된 IP 주소나 보안 그룹

웹 서버와 DB에는 서로 다른 보안 그룹을 사용한다.

| 대상 | 인바운드 | 출발지 |
| :--- | :--- | :--- |
| 웹 EC2 | SSH 22 | 관리자 공인 IP 또는 VPN 대역 |
| 웹 EC2 | HTTP 80 | 인터넷 전체 `0.0.0.0/0` |
| 웹 EC2 | HTTPS 443 | 인터넷 전체 `0.0.0.0/0` |
| RDS | PostgreSQL 5432 | 웹 EC2의 보안 그룹 |

IPv6를 실제로 구성했다면 HTTP와 HTTPS에 `::/0`도 추가한다. IPv6를 사용하지 않는 환경에서는 불필요한 규칙을 미리 열지 않는다.

RDS 규칙의 출발지를 EC2의 특정 IP가 아니라 **웹 보안 그룹**으로 지정하는 것이 중요하다. 그러면 웹 보안 그룹을 가진 리소스만 DB에 연결할 수 있고, EC2의 사설 IP가 바뀌어도 규칙을 고칠 필요가 없다.

### SSH와 Session Manager는 다른 선택지다

직접 SSH로 접속한다면 22번 포트를 관리자 IP에만 허용한다. AWS Systems Manager Session Manager를 사용하면 인바운드 22번 포트를 열지 않고도 IAM 권한으로 서버에 접속할 수 있다. 이 경우에는 인스턴스 역할, SSM Agent, 외부 서비스로 나가는 네트워크 경로가 필요하다.

### 실제로 자주 겪는 문제: SSH 타임아웃

SSH 키가 맞는데도 연결이 계속 시간 초과된 적이 있었다. 원인은 보안 그룹의 SSH 출발지와 현재 사용 중인 공인 IP가 달랐기 때문이다. 인터넷 회선이나 장소가 바뀌면 관리자 공인 IP도 달라질 수 있다.

```text
연결 시간 초과
  → EC2가 실행 중인가?
  → 주소가 맞는가?
  → 보안 그룹의 22번 출발지가 현재 관리자 IP인가?
  → 서브넷 라우팅과 Network ACL은 정상인가?
```

키 인증은 “누구인지” 확인하고, 보안 그룹은 “어디에서 들어올 수 있는지” 제한한다. 둘 중 하나만 맞아도 되는 것이 아니다.

## 5. 비공개 RDS PostgreSQL을 준비한다

### Endpoint와 포트란?

RDS를 만들면 AWS가 `example.xxxxxx.region.rds.amazonaws.com` 같은 **endpoint**를 제공한다. endpoint는 데이터베이스 서버의 DNS 이름이고, PostgreSQL의 기본 포트는 `5432`다.

애플리케이션에는 다음 정보가 필요하다.

- endpoint
- 포트
- DB 이름
- 사용자 이름
- 비밀번호

RDS를 생성할 때는 다음 항목도 확인한다.

- 애플리케이션과 호환되는 PostgreSQL 버전
- 비공개 DB subnet group
- Public access 비활성화
- 웹 보안 그룹만 허용하는 DB 보안 그룹
- 백업 보존 기간과 유지보수 창
- 삭제 방지와 최종 스냅샷 정책
- 예상 트래픽에 맞는 인스턴스와 스토리지 크기

### 앱을 배포하기 전에 네트워크부터 검사한다

Rails까지 한 번에 실행하면 DB 접속 실패의 원인을 찾기 어렵다. 먼저 EC2에서 RDS endpoint의 5432번 포트에 도달할 수 있는지만 검사한다. `nc`는 TCP 연결 가능 여부를 확인하는 netcat 명령이며 AMI에 따라 별도 설치가 필요할 수 있다.

```bash
nc -zv "$DATABASE_HOST" "${DATABASE_PORT:-5432}"
```

- `-z`: 데이터를 보내지 않고 포트가 열렸는지만 검사한다.
- `-v`: 확인 과정을 자세히 표시한다.
- `${DATABASE_PORT:-5432}`: 환경 변수가 없으면 5432를 사용한다.

연결 성공은 적어도 VPC 내부 DNS, 라우팅, EC2 보안 그룹, RDS 보안 그룹이 연결 가능한 상태라는 뜻이다. DB 사용자나 비밀번호가 맞다는 의미는 아니므로 실제 로그인 검사는 별도로 해야 한다.

이렇게 작은 단위로 확인해 두면 이후 Rails에서 오류가 날 때 네트워크보다 DB 설정과 비밀값을 먼저 살펴볼 수 있다.

## 6. EC2와 Docker를 준비한다

### Docker가 필요한 이유

서버에 Ruby, Node.js, 각종 시스템 라이브러리를 직접 설치하면 개발 환경과 운영 환경이 달라지기 쉽다. Docker는 이 실행 환경을 **이미지(image)**로 만들어 같은 단위로 실행한다.

- **Dockerfile**: 앱을 어떻게 포장할지 적은 조리법
- **이미지**: Rails 코드, Ruby, gem, 시스템 패키지를 묶은 결과물
- **컨테이너**: 이미지를 실제로 실행한 프로세스
- **레지스트리**: 완성된 이미지를 보관하고 전달하는 저장소

Amazon Linux 계열 EC2에서 Docker를 준비하는 한 예는 다음과 같다. AMI에 따라 패키지 관리자와 기본 사용자 이름이 다르므로 현재 운영체제 문서를 확인한다.

```bash
sudo dnf install -y docker
sudo systemctl enable --now docker
sudo usermod -aG docker ec2-user
```

- `enable --now`는 Docker를 지금 시작하고 재부팅 뒤에도 자동 실행한다.
- `-aG docker`는 기존 보조 그룹을 유지하면서 사용자를 `docker` 그룹에 추가한다.
- 그룹 변경은 새 로그인 세션부터 반영되므로 SSH를 끊고 다시 접속한다.

```bash
docker ps
```

오류 없이 컨테이너 목록이 보이면 Docker 제어가 가능하다. 단, `docker` 그룹은 사실상 root 수준의 권한을 제공한다. 신뢰할 수 있는 배포 사용자에게만 부여하고, 더 엄격한 격리가 필요하면 rootless Docker 같은 방식을 검토한다.

### CPU 아키텍처를 맞춘다

Docker 이미지도 실행할 CPU 종류가 맞아야 한다.

- Apple Silicon과 AWS Graviton: `arm64`
- 일반적인 Intel·AMD 서버: `amd64`

개발 장비와 EC2가 모두 ARM64였을 때는 다음처럼 별도 에뮬레이션 없이 이미지를 만들 수 있었다.

```bash
docker build --platform linux/arm64 -t example-app:local .
```

서로 다른 아키텍처를 사용해도 다중 플랫폼 빌드가 가능하지만 빌드 시간이 늘거나 일부 라이브러리에서 문제가 생길 수 있다. EC2를 고르기 전에 사용하는 gem과 시스템 패키지가 해당 아키텍처를 지원하는지 확인한다.

### Docker가 앱을 느리게 만들까?

Linux의 Docker 컨테이너는 호스트 커널을 공유하므로 가상머신처럼 운영체제를 하나 더 실행하지 않는다. 일반적인 Rails 서비스에서는 Docker 자체보다 DB 쿼리, 네트워크 지연, Puma 설정, 이미지 처리, 캐시 여부가 성능에 더 큰 영향을 주는 경우가 많다.

Docker의 주된 목적은 성능 향상이 아니라 환경 재현, 격리, 배포와 롤백 자동화다. 대신 이미지 빌드 시간, 디스크 사용량, 로그와 업로드 파일의 영속성을 따로 관리해야 한다.

## 7. Kamal로 배포 흐름을 자동화한다

Kamal은 Rails의 일부가 아니라 Docker 컨테이너를 서버에 배포하는 도구다. Docker 이미지로 만들 수 있다면 Rails 외의 앱에도 사용할 수 있다.

```text
개발 컴퓨터의 Git 소스와 Dockerfile
             │ 이미지 빌드
             ▼
        Docker Registry
             │ 이미지 전달
             ▼
Kamal ── SSH ── EC2의 Docker
                    │
                    ▼
              Rails 컨테이너
```

Kamal이 주로 담당하는 작업은 다음과 같다.

- 커밋된 소스로 이미지 빌드
- 레지스트리를 통한 이미지 전달
- SSH로 원격 Docker 명령 실행
- 환경 변수와 비밀값 주입
- 영구 볼륨 연결
- Kamal Proxy 실행
- 헬스 체크 후 새 버전으로 트래픽 전환
- 이전 이미지로 롤백

Kamal은 VPC, 보안 그룹, RDS를 대신 만들지는 않는다. AWS 네트워크가 먼저 연결되어 있어야 한다.

### 로컬 레지스트리와 중앙 레지스트리

Kamal 설정에서 레지스트리를 `localhost`로 지정하면 배포 컴퓨터에 로컬 레지스트리를 실행할 수 있다.

```yaml
registry:
  server: localhost:5555
```

배포 중에는 Kamal이 SSH를 통해 이미지를 서버로 전달한다. 배포가 끝나고 EC2가 이미지를 내려받은 뒤에는 개발 컴퓨터와 로컬 레지스트리를 꺼도 실행 중인 서비스는 계속 동작한다.

한 사람이 한 대의 서버에 배포할 때는 간단하지만, CI/CD나 여러 배포 담당자·서버를 사용한다면 AWS ECR 같은 중앙 레지스트리가 더 적합하다.

## 8. Kamal Proxy, Thruster, Puma를 구분한다

Rails 8의 기본 Docker 구성에서는 이름이 비슷한 여러 구성요소가 요청 경로에 등장한다.

```text
[사용자]
   │ EC2의 80/443
   ▼
[Kamal Proxy 컨테이너]
   │ Docker 내부 네트워크
   ▼
[Rails 앱 컨테이너]
   ├── Thruster: 정적 파일 캐싱·압축과 전송 보조
   ├── Puma: Ruby 요청을 처리하는 웹 서버
   └── Rails: 애플리케이션 코드
```

- **리버스 프록시**는 사용자의 요청을 먼저 받은 뒤 내부 앱 서버로 전달하는 프로그램이다.
- **Puma**는 Ruby 프로세스 안에서 Rails 요청을 처리한다.
- **Thruster**는 Rails 컨테이너 안에서 Puma 앞에 실행되어 정적 파일 전송 등을 보조한다.
- **Kamal Proxy**는 Rails 컨테이너와 별도로 실행되며 새 버전 전환과 외부 HTTPS를 담당한다.

새 버전 배포 시에는 기존 컨테이너를 먼저 끄지 않는다.

```text
기존 버전이 요청 처리
  → 새 컨테이너 시작
  → /up 헬스 체크 성공
  → Proxy가 새 버전으로 요청 전환
  → 기존 컨테이너 종료
```

이 과정이 배포 중단 시간을 줄인다.

## 9. Rails와 RDS를 환경 변수로 연결한다

개발 DB와 운영 DB의 주소는 서로 다르다. 주소와 비밀번호를 코드에 직접 적는 대신 환경 변수를 사용하면 동일한 코드로 여러 환경을 실행할 수 있다.

아래는 **운영 DB를 하나만 사용하는 Rails 앱의 최소 예시**다.

```yaml
# config/database.yml
production:
  primary:
    adapter: postgresql
    host: <%= ENV.fetch("DATABASE_HOST") %>
    port: <%= ENV.fetch("DATABASE_PORT", 5432) %>
    database: <%= ENV.fetch("DATABASE_NAME") %>
    username: <%= ENV.fetch("DATABASE_USERNAME") %>
    password: <%= ENV.fetch("DATABASE_PASSWORD") %>
```

Rails 8 앱이 Solid Cache, Solid Queue, Solid Cable용 데이터베이스를 분리했다면 기존의 `primary`, `cache`, `queue`, `cable` 구성을 유지하고 각각 필요한 DB 이름을 지정해야 한다. 이 다중 DB 구조는 Docker나 Kamal 때문이 아니라 Rails 애플리케이션 구성에 따른 것이다.

Kamal은 `config/deploy.yml`의 `env`를 통해 값을 컨테이너에 전달한다.

```text
RDS endpoint와 port
  → Kamal 일반 환경 변수
  → Docker 컨테이너 환경 변수
  → Rails database.yml
```

## 10. 비밀번호와 master key를 안전하게 전달한다

endpoint와 포트는 보통 일반 설정으로 다뤄도 되지만 DB 비밀번호와 `RAILS_MASTER_KEY`는 비밀값이다. Git, 터미널 출력, 배포 로그에 평문으로 남기지 않는다.

Rails credentials를 사용하는 한 가지 흐름은 다음과 같다.

```text
실제 DB 비밀번호
  → config/credentials.yml.enc에 암호화
  → master key로 배포 시 복호화
  → .kamal/secrets가 값을 읽음
  → 컨테이너의 DATABASE_PASSWORD로 전달
```

- `config/credentials.yml.enc`: 암호화되어 있어 Git에 커밋할 수 있다.
- `config/master.key`: 복호화 열쇠이므로 Git에 커밋하지 않는다.
- `.kamal/secrets`: 비밀번호를 직접 적기보다 credentials나 비밀 저장소에서 읽는다.

검증할 때도 값을 화면에 출력하지 않고 존재 여부와 명령 성공 여부만 확인한다.

```bash
bin/rails credentials:fetch database_password >/dev/null \
  && echo CREDENTIAL_FOUND

bin/kamal config >/dev/null \
  && echo KAMAL_CONFIG_OK
```

`>/dev/null`은 실제 값을 화면에 표시하지 않고 버린다. 성공 메시지만으로 필요한 값이 존재하는지 확인할 수 있다.

## 11. 최소 Kamal 설정에서 바꿔야 할 값

다음은 전체 설정 파일이 아니라 핵심 구조를 보여 주는 예시다. 대문자 자리표시자는 자신의 환경에 맞게 교체한다.

```yaml
service: example-app
image: example-app

servers:
  web:
    - SERVER_ADDRESS

ssh:
  user: ec2-user

builder:
  arch: arm64 # x86_64 EC2라면 amd64

registry:
  server: localhost:5555

env:
  clear:
    DATABASE_HOST: RDS_ENDPOINT
    DATABASE_PORT: 5432
    DATABASE_NAME: example_production
    DATABASE_USERNAME: app_user
  secret:
    - RAILS_MASTER_KEY
    - DATABASE_PASSWORD
```

실제 Kamal 버전과 프로젝트의 기존 `deploy.yml` 구조가 우선이다. 설정을 통째로 덮어쓰기보다 필요한 항목을 기존 파일에 반영하고 `bin/kamal config`로 해석 결과를 확인한다.

## 12. 첫 배포 전 체크리스트

- [ ] root 계정과 작업용 IAM 계정에 MFA를 적용했다.
- [ ] 예산 알림과 실습 종료 후 정리 기준을 만들었다.
- [ ] RDS는 비공개 서브넷에 있고 Public access가 꺼져 있다.
- [ ] RDS의 5432번은 웹 보안 그룹에서만 접근할 수 있다.
- [ ] 서버의 SSH 또는 Session Manager 접근을 확인했다.
- [ ] EC2에서 RDS endpoint:5432 연결을 확인했다.
- [ ] EC2에서 `docker ps`가 실행된다.
- [ ] 서버 CPU와 이미지 아키텍처가 일치한다.
- [ ] DB 비밀번호와 master key가 Git에 남아 있지 않다.
- [ ] `/up` 헬스 체크가 인증 없이 성공 응답을 반환한다.
- [ ] `bin/kamal config`가 비밀값을 노출하지 않고 성공한다.
- [ ] 로그 확인과 롤백 방법을 적어 두었다.

첫 서버 준비와 첫 배포에는 일반적으로 다음 명령을 사용한다.

```bash
bin/kamal setup
```

이후 애플리케이션 변경을 배포할 때는 보통 `bin/kamal deploy`를 사용한다. 명령은 사용하는 Kamal 버전에 따라 달라질 수 있으므로 `bin/kamal help`로 확인한다.

```text
첫 HTTP 배포와 /up 확인
  → Elastic IP로 서버 주소 고정
  → 도메인 연결과 HTTPS 적용
  → S3·백업·로그·모니터링 보강
```

[다음 글: AWS EC2의 고정 주소와 Kamal HTTPS 적용하기]({% post_url 2026-08-24-aws-restart-elastic-ip-https %})

## 참고 문서

- [AWS: EC2와 RDS 연결 튜토리얼](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/tutorial-connect-ec2-instance-to-rds-database.html)
- [AWS: RDS DB 인스턴스 생성](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_CreateDBInstance.html)
- [AWS: Systems Manager Session Manager](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager.html)
- [Docker: Linux 설치 후 설정](https://docs.docker.com/engine/install/linux-postinstall/)
- [Kamal: Docker Registry 설정](https://kamal-deploy.org/docs/configuration/docker-registry/)
