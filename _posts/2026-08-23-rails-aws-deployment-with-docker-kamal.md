---
title: "Rails 앱 AWS 배포 준비: VPC, RDS, EC2, Docker와 Kamal"
date: 2026-08-23 23:59:00 +0900
categories: [DevOps, AWS]
tags: [Rails, AWS, VPC, EC2, RDS, PostgreSQL, Docker, Kamal, ReverseProxy, Deployment]
---

# 🚀 Rails 앱 AWS 배포 준비: VPC, RDS, EC2, Docker와 Kamal

> **한 줄 요약**:  
> Rails 앱을 AWS에 배포하기 위해 VPC 안에 공개 EC2와 비공개 RDS를 구성하고, Docker 이미지와 Kamal을 이용해 재현 가능한 배포 경로를 준비했다. 현재 인프라와 설정 검증은 끝났고, 첫 `bin/kamal setup` 실행만 남아 있다.

---

## 1. 📌 최종적으로 만들려는 구조

현재는 비용과 학습 범위를 줄이기 위해 웹 서버 한 대로 시작한다.

```text
[사용자 브라우저]
        │ HTTP 80 / 향후 HTTPS 443
        ▼
[공개 서브넷의 EC2]
        │
        ├── Docker: kamal-proxy 컨테이너
        │            │
        │            ▼
        └── Docker: HJMemo Rails 컨테이너
                     │
                     │ PostgreSQL 5432
                     ▼
             [비공개 서브넷의 RDS]
```

배포할 때만 개발 Mac이 다음 경로에 참여한다.

```text
[Mac의 Docker]
      │ 이미지 빌드
      ▼
[로컬 Docker Registry]
      │ Kamal이 SSH 터널 생성
      ▼
[EC2의 Docker]
```

배포가 끝난 뒤에는 Mac과 로컬 레지스트리를 꺼도 EC2의 서비스는 계속 실행된다.

---

## 2. 🔐 AWS 계정 보안과 비용 안전장치

배포 리소스를 만들기 전에 계정부터 보호했다.

- root 계정에 MFA(Multi-Factor Authentication) 적용
- 평상시 사용할 관리자 IAM 사용자 생성
- IAM 관리자에게도 MFA 적용
- IAM 사용자의 결제 정보 접근 활성화
- 월 비용 예산 알림 설정

### 왜 root 계정을 계속 사용하면 안 될까?

root는 계정 전체를 삭제하거나 결제·보안 설정까지 바꿀 수 있는 최상위 계정이다. 일상적인 작업에서는 IAM 사용자를 쓰고, root는 비상시에만 사용하는 것이 안전하다.

### 비용 정책

이번 구성은 AWS 크레딧 안에서 배포를 경험하는 것이 목표다. 학습 단계에서는 다음 고정 비용과 복잡도를 피했다.

- NAT Gateway 없음
- Application Load Balancer 없음
- 다중 EC2 없음
- RDS Multi-AZ 없음

---

## 3. 🌐 VPC와 서브넷

### VPC란?

VPC(Virtual Private Cloud)는 AWS 안에 만드는 프로젝트 전용 네트워크다. EC2와 RDS가 어느 주소를 사용하고, 인터넷과 연결되는지, 서로 통신할 수 있는지를 정한다.

```text
[HJMemo VPC]
├── public1  : 현재 EC2 배치
├── public2  : 향후 ALB 또는 두 번째 EC2 배치
├── private1 : RDS용
└── private2 : RDS subnet group의 두 번째 가용 영역
```

공개 서브넷은 Internet Gateway로 가는 경로를 가진다. 공개 IP를 받은 EC2는 이 경로를 통해 인터넷과 통신할 수 있다.

비공개 서브넷은 인터넷에서 직접 들어오는 경로가 없다. RDS는 여기에 배치해 인터넷에 DB 포트를 노출하지 않았다.

### NAT Gateway를 만들지 않은 이유

NAT Gateway는 비공개 서브넷의 서버가 외부 API나 패키지 저장소로 나갈 수 있게 해준다. 현재 웹 EC2는 공개 서브넷에 있고, RDS는 외부 인터넷에 나갈 필요가 없으므로 NAT가 필요하지 않다.

NAT는 단순히 서버가 외부 API를 호출한다고 무조건 붙이는 것이 아니다. **인터넷으로 나가야 하는 서버가 비공개 서브넷에 있을 때** 필요하다.

---

## 4. 🛡️ 보안 그룹: 인바운드와 아웃바운드

보안 그룹은 EC2와 RDS 앞에 놓인 상태 기반 가상 방화벽이다.

- **인바운드(Inbound)**: 외부에서 리소스로 들어오는 요청
- **아웃바운드(Outbound)**: 리소스가 외부로 보내는 요청

### 웹 보안 그룹

```text
SSH   22  ← 현재 관리자의 공인 IP만
HTTP  80  ← 0.0.0.0/0
HTTPS 443 ← 0.0.0.0/0
```

SSH 키가 있더라도 22번 포트를 `0.0.0.0/0`으로 계속 열지 않았다. 키 인증은 신원을 확인하는 두 번째 방어선이고, 보안 그룹은 불필요한 접속 시도 자체를 서버 앞에서 차단하는 첫 번째 방어선이다.

관리자의 공인 IP가 바뀌면 SSH 규칙의 `My IP`를 갱신해야 한다.

### DB 보안 그룹

```text
PostgreSQL 5432 ← 웹 보안 그룹을 가진 리소스만
```

DB 규칙의 출발지를 특정 IP 대신 웹 보안 그룹으로 지정했다. 그 결과 인터넷에서는 RDS에 접근할 수 없고, HJMemo EC2만 RDS에 연결할 수 있다.

---

## 5. 🐘 비공개 RDS PostgreSQL

운영 데이터베이스는 Amazon RDS PostgreSQL로 만들었다.

- ARM 기반 소형 인스턴스 사용
- 20GiB gp3 스토리지
- Public access 비활성화
- 두 비공개 서브넷으로 DB subnet group 구성
- 운영 DB 이름: `hjmemo_production`
- PostgreSQL 기본 포트: `5432`

DB subnet group에 두 가용 영역의 서브넷을 넣었다고 해서 현재 DB가 자동으로 Multi-AZ가 되는 것은 아니다. 현재는 비용을 줄인 Single-AZ 인스턴스이고, 두 번째 서브넷은 향후 배치 선택지와 확장 기반을 제공한다.

EC2에서 RDS Endpoint의 5432번 포트로 연결을 검사했고 다음 결과를 확인했다.

```text
RDS_REACHABLE
```

이는 다음 세 가지가 정상이라는 뜻이다.

1. VPC 내부 DNS와 네트워크 경로
2. EC2 웹 보안 그룹
3. RDS DB 보안 그룹

---

## 6. 🖥️ 공개 EC2 웹 서버

웹 서버는 다음 구성으로 만들었다.

- Amazon Linux 2023
- ARM64(`aarch64`)
- Graviton 계열 `t4g.small`
- 공개 서브넷 `public1`
- 20GiB gp3
- 퍼블릭 IPv4 사용
- SSH 키 페어 사용
- 인스턴스 상태 검사 `3/3` 통과

처음에는 SSH 연결이 시간 초과됐다. 웹 보안 그룹의 SSH 출발지 IP가 현재 공인 IP와 다르다는 것을 확인하고 `My IP`를 갱신하여 해결했다.

EC2를 중지했다가 시작하면 자동 할당 퍼블릭 IP가 바뀔 수 있다. 이 경우 다음 두 설정도 갱신해야 한다.

- 웹 보안 그룹의 SSH 출발지 IP
- `config/deploy.yml`의 대상 서버 IP

향후 도메인과 안정적인 운영 주소가 필요하면 Elastic IP 또는 ALB 구성을 검토한다.

---

## 7. 🐳 EC2와 Mac에 Docker 준비

### EC2 Docker

Amazon Linux 2023에서 Docker를 설치하고 서비스를 시작했다.

```bash
sudo dnf install -y docker
sudo systemctl enable --now docker
sudo usermod -aG docker ec2-user
```

`-aG`는 기존 보조 그룹을 유지한 채 `docker` 그룹을 추가한다.

- `-G docker`: 보조 그룹을 지정
- `-a`: 기존 그룹을 지우지 않고 append

재로그인 후 `sudo` 없이 `docker ps`가 실행되는 것을 확인했다.

### Mac Docker

로컬 Mac에도 Docker Desktop을 설치했다.

```text
Mac Docker: linux/arm64
EC2:        aarch64(ARM64)
```

Mac과 EC2의 CPU 구조가 같기 때문에 QEMU 에뮬레이션 없이 ARM64 이미지를 만들 수 있다.

프로젝트의 `Dockerfile`로 이미지 빌드도 성공했다.

```bash
docker build --platform linux/arm64 -t hjmemo:local .
```

---

## 8. 📦 이미지, 컨테이너, 레지스트리

세 용어를 구분해야 한다.

- **Dockerfile**: 앱을 어떻게 포장할지 적은 조리법
- **Docker image**: Rails 코드, Ruby, gem, 시스템 패키지를 포장한 실행 가능한 결과물
- **Docker container**: 이미지를 실제로 실행한 프로세스
- **Docker registry**: 완성된 이미지를 보관하고 전달하는 창고

GitHub가 소스 코드 저장소라면 Docker Registry는 완성된 이미지 저장소다.

현재는 Docker Hub나 AWS ECR 대신 Kamal 2의 로컬 레지스트리를 사용한다.

```text
127.0.0.1:5555 → kamal-docker-registry
```

- Mac 안에서만 접근 가능
- 인터넷에 공개되지 않음
- 별도 레지스트리 계정이나 AWS 비용 없음
- Kamal이 배포 중 SSH 터널로 EC2까지 연결

다중 서버나 CI/CD 환경으로 확장할 때는 ECR 같은 중앙 레지스트리가 더 편리하다.

---

## 9. 🚢 Kamal의 역할

Kamal은 Rails 자체 구성요소는 아니지만 Rails 생태계에서 만들어진 Docker 배포 도구다. 원래 Rails를 위해 만들어졌지만 Docker 이미지로 만들 수 있는 다른 웹 앱에도 사용할 수 있다.

```text
[Rails 소스 + Dockerfile]
          │
          ▼
[로컬 Docker가 이미지 빌드]
          │
          ▼
[Registry에 이미지 저장]
          │
          ▼
[Kamal이 SSH로 EC2 Docker 제어]
          │
          ▼
[Rails 컨테이너 실행]
```

Kamal이 담당하는 작업:

- 커밋된 Git 소스로 이미지 빌드
- 이미지 레지스트리 전달
- SSH를 통한 원격 Docker 명령 실행
- 환경 변수와 비밀값 주입
- 영구 볼륨 연결
- `kamal-proxy` 실행
- `/up` 헬스 체크
- 새 버전 전환과 이전 컨테이너 정리
- 이전 이미지로 롤백

Kamal의 SSH 연결을 통해 EC2에서 `docker ps`를 실행했고 성공했다.

```bash
bin/kamal server exec "docker ps"
```

---

## 10. 🔁 Kamal Proxy, Thruster, Puma의 차이

Nginx는 설치하지 않았다. 현재 Rails 8 + Kamal 구성에서는 역할이 다음처럼 나뉜다.

```text
[사용자]
   │ EC2 80/443
   ▼
[kamal-proxy 컨테이너]
   │ Docker 내부 네트워크
   ▼
[Rails 앱 컨테이너]
   │
   ├── Thruster: 정적 파일 캐싱·압축·전송 가속
   ├── Puma: Ruby 웹 서버
   └── Rails: 실제 애플리케이션
```

`kamal-proxy`는 Rails 컨테이너 안이 아니라 EC2의 Docker에서 별도 컨테이너로 실행된다.

새 버전 배포 흐름:

```text
Rails v1이 요청 처리
       ↓
Rails v2 컨테이너 시작
       ↓
/up 헬스 체크 성공
       ↓
Proxy가 v1에서 v2로 요청 전환
       ↓
v1 컨테이너 종료
```

이 구조가 배포 중단 시간을 최소화한다.

프로젝트의 Dockerfile은 앱 컨테이너 안에서 다음 명령으로 시작한다.

```dockerfile
CMD ["./bin/thrust", "./bin/rails", "server"]
```

---

## 11. 🗄️ Rails와 RDS 연결 설정

운영 앱은 DB 이름, 사용자, 암호, host, port를 알아야 한다. 이것은 Docker나 Kamal에만 필요한 개념이 아니라 외부 DB를 사용하는 일반적인 배포 요구사항이다.

구현 방식은 도구별로 달라진다.

| 구분 | 현재 적용 방식 |
| :--- | :--- |
| 일반 배포 | DB 연결 정보와 비밀값 필요 |
| Rails | `config/database.yml`에서 환경 변수 읽기 |
| AWS/RDS | RDS Endpoint와 5432 포트 사용 |
| Kamal | `config/deploy.yml`의 `env`로 전달 |
| Docker | 컨테이너 환경 변수로 주입 |

운영 DB 설정은 다음 환경 변수를 읽는다.

```text
HJMEMO_DATABASE_HOST
HJMEMO_DATABASE_PORT
HJMEMO_DATABASE_PASSWORD
```

Rails 앱은 Solid Cache, Solid Queue, Solid Cable을 사용하므로 운영 DB 구성이 다음처럼 나뉜다.

```text
hjmemo_production
hjmemo_production_cache
hjmemo_production_queue
hjmemo_production_cable
```

이 다중 DB 구성은 Docker나 Kamal 때문이 아니라 현재 Rails 8 앱 구성 때문이다.

---

## 12. 🔑 운영 비밀값 관리

RDS 암호를 `database.yml`, `deploy.yml`, `.kamal/secrets`에 평문으로 적지 않았다.

```text
[실제 RDS 암호]
       │ Rails credentials로 암호화
       ▼
[config/credentials.yml.enc]
       │ 배포 시 master key로 복호화
       ▼
[.kamal/secrets]
       │ Kamal이 컨테이너에 전달
       ▼
[HJMEMO_DATABASE_PASSWORD]
```

- `config/credentials.yml.enc`: 암호화되어 Git 커밋 가능
- `config/master.key`: 복호화 열쇠이므로 Git에서 제외
- `.kamal/secrets`: 암호 자체가 아닌 credentials 조회 명령만 포함

비밀값을 출력하지 않고 존재 여부만 확인했다.

```bash
bin/rails credentials:fetch hjmemo_database_password >/dev/null && echo CREDENTIAL_FOUND
```

Kamal 설정 역시 출력값을 폐기한 뒤 성공 여부만 확인했다.

```bash
bin/kamal config >/dev/null && echo KAMAL_CONFIG_OK
```

---

## 13. ⚙️ 현재 Kamal 설정 요약

`config/deploy.yml`에는 다음 항목을 반영했다.

- 서비스 이름: `hjmemo`
- 대상: 단일 EC2
- SSH 사용자: `ec2-user`
- 지정한 개인 키만 사용
- 빌드 아키텍처: `arm64`
- 레지스트리: `localhost:5555`
- DB host와 port: 일반 환경 변수
- Rails master key와 DB password: secret 환경 변수
- Rails 로컬 저장소: Docker named volume 연결

설정 변경은 다음 커밋으로 확정했다.

```text
7d3c40e Configure AWS deployment
```

Kamal이 기본적으로 커밋된 Git 상태를 복제해 빌드하기 때문에 배포 설정을 커밋했다. Rails 자동 테스트도 통과했다.

---

## 14. ⚡ Docker 실행은 직접 Rails 실행보다 느릴까?

EC2 Linux에서 Docker 컨테이너는 호스트 Linux 커널을 공유한다. 가상머신처럼 운영체제를 하나 더 실행하지 않으므로 CPU와 메모리 성능 차이는 일반적으로 크지 않다.

Rails 서비스에서 더 큰 영향을 주는 요소:

- DB 쿼리와 RDS 네트워크 지연
- Puma 스레드와 프로세스 수
- EC2 CPU와 메모리
- 이미지 처리
- 캐시 사용 여부

Docker의 주된 목적은 성능 향상이 아니라 **환경 재현, 격리, 배포와 롤백 자동화**다.

장점:

- Mac과 EC2에서 동일한 실행 환경
- EC2에 Ruby와 각 라이브러리를 직접 설치할 필요가 없음
- 같은 이미지를 여러 서버에 배포 가능
- 새 버전 전환과 롤백이 쉬움

단점:

- Docker 네트워크와 이미지 관리 개념 추가
- 이미지 빌드·전송 시간과 디스크 사용
- 로그와 업로드 파일 영속성 별도 설계
- ARM64/AMD64 아키텍처 관리

---

## 15. 🖧 EC2가 여러 대라면?

Kamal 설정에 여러 서버를 선언하면 같은 이미지를 모두 배포할 수 있다.

```yaml
servers:
  web:
    - WEB_SERVER_1
    - WEB_SERVER_2
```

그러나 Kamal은 로드 밸런서가 아니다. 사용자의 요청을 여러 서버에 분산하려면 AWS Application Load Balancer 같은 별도 구성요소가 필요하다.

```text
                   ┌── EC2 web 1 ──┐
[사용자] → [ALB] ──┤                ├──→ 공용 RDS
                   └── EC2 web 2 ──┘
                            └──────────→ 공용 S3
```

여러 웹 서버에서는 로컬 업로드 볼륨을 공유할 수 없으므로 S3 같은 객체 저장소가 필수적이다. 현재 단일 서버 구성은 첫 배포의 원리를 배우고 비용을 줄이기 위한 선택이다.

---

## 16. ✅ 현재 체크포인트

완료:

- AWS 계정 보안 및 예산 알림
- VPC와 공개·비공개 서브넷
- 웹·DB 보안 그룹
- 비공개 RDS PostgreSQL
- 공개 EC2와 SSH
- EC2 Docker
- Mac Docker Desktop
- ARM64 Rails 이미지 빌드
- 로컬 Docker Registry
- Kamal SSH 연결
- EC2 → RDS 네트워크 확인
- Rails DB 환경 변수 설정
- Rails credentials와 Kamal secrets 연결
- Kamal 설정 검증
- Git 커밋
- Rails 자동 테스트

아직 하지 않은 것:

- `bin/kamal setup` 첫 실행
- EC2의 `kamal-proxy`와 Rails 컨테이너
- 실제 운영 DB 마이그레이션
- HTTP 페이지와 `/up` 확인
- 도메인과 HTTPS
- S3 Active Storage
- 실제 사용자 흐름 점검
- 백업과 모니터링

---

## 17. ▶️ 바로 다음 작업

```bash
bin/kamal setup
```

이 명령은 다음 작업을 수행한다.

1. 커밋된 Rails 소스로 ARM64 이미지 빌드
2. 로컬 레지스트리에 이미지 저장
3. SSH 터널을 통해 EC2로 이미지 전달
4. `kamal-proxy` 컨테이너 실행
5. Rails 컨테이너 실행
6. RDS DB 준비 및 마이그레이션
7. `/up` 헬스 체크
8. EC2의 HTTP 80번 포트로 서비스 공개

배포 성공 후에는 EC2의 퍼블릭 IP에서 `/up`과 비민감 화면만 확인한다. 아직 HTTPS가 없으므로 실제 비밀번호로 회원가입이나 로그인하지 않는다.

이후 진행 순서:

```text
첫 HTTP 배포 확인
  → 도메인과 HTTPS
  → S3 이미지 영구 저장
  → 운영 사용자 흐름 점검
  → 백업·로그·오류 추적
```
