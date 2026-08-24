---
title: "Rails 앱 AWS 배포 재개: Elastic IP와 Kamal HTTPS"
date: 2026-08-24 23:59:00 +0900
categories: [DevOps, AWS]
tags: [Rails, AWS, EC2, RDS, ElasticIP, Docker, Kamal, KamalProxy, HTTPS, TLS, LetsEncrypt, DNS, sslip]
---

# 🔐 Rails 앱 AWS 배포 재개: Elastic IP와 Kamal HTTPS

> **한 줄 요약**:  
> 중지했던 EC2와 RDS를 복구한 뒤 Elastic IP로 주소를 고정하고, `sslip.io`, Kamal Proxy, Let's Encrypt를 이용해 HJMemo에 HTTPS를 적용했다.

최종 접속 주소:

```text
https://3-34-34-46.sslip.io
```

---

## 1. 🔄 EC2와 RDS 다시 시작

비용 절약을 위해 중지했던 자원을 다음 순서로 다시 시작했다.

1. RDS PostgreSQL을 시작하고 상태가 `Available`이 될 때까지 기다렸다.
2. EC2 `hjmemo-web`을 시작했다.
3. EC2에 새로 할당된 자동 공인 IP를 확인했다.

이때 자동 공인 IP는 `54.180.236.57`이었다.

EC2가 시작되자 Docker와 기존 컨테이너도 다시 실행됐다.

- `kamal-proxy`
- `hjmemo-web-...` Rails 컨테이너

새로운 이미지를 다시 배포하지 않아도 기존 컨테이너가 자동으로 복구됐다.

---

## 2. ✅ 애플리케이션 복구 확인

Rails의 상태 확인 주소에 요청을 보냈다.

```bash
curl -I http://54.180.236.57/up
```

다음 응답을 받아 애플리케이션이 정상적으로 동작하고 있음을 확인했다.

```text
HTTP/1.1 200 OK
```

브라우저에서도 앱 화면이 정상적으로 표시됐다.

이때 요청 흐름은 다음과 같다.

```text
브라우저
→ EC2 공인 IP
→ kamal-proxy
→ Rails Docker 컨테이너
→ RDS PostgreSQL
```

---

## 3. 🌐 자동 공인 IP의 문제 확인

EC2를 중지했다가 시작하면 자동 공인 IP가 바뀔 수 있다. 따라서 처음에는 `config/deploy.yml`의 Kamal 배포 대상도 새 주소로 바꿔야 했다.

```yaml
servers:
  web:
    - 54.180.236.57
```

주소가 바뀌면 다음 작업을 반복해야 한다.

- Kamal 배포 서버 주소 변경
- SSH 접속 주소 변경
- 도메인이 있다면 DNS 레코드 변경

안정적인 배포 주소를 사용하기 위해 Elastic IP를 연결하기로 했다.

---

## 4. 📍 Elastic IP 연결

AWS에서 Elastic IP를 할당하고 `hjmemo-web` EC2에 연결했다.

```text
Elastic IP: 3.34.34.46
```

Elastic IP는 EC2를 다시 시작해도 유지되는 고정 공인 IPv4 주소다. 연결되면서 기존 자동 공인 IP는 반납되고 Elastic IP가 EC2의 공인 주소가 됐다.

EC2 실행 중에는 기존 자동 공인 IP 한 개가 Elastic IP 한 개로 교체되므로 공인 IP 개수가 늘어나지 않는다. 다만 EC2를 중지해도 Elastic IP를 계속 보유하면 공인 IPv4 비용은 계속 발생한다.

Kamal 설정도 고정 IP로 변경했다.

```yaml
servers:
  web:
    - 3.34.34.46
```

다음 두 경로를 모두 확인했다.

```text
Kamal → SSH → Elastic IP → EC2
브라우저 → HTTP → Elastic IP → Kamal Proxy → Rails
```

변경사항은 다음 커밋으로 기록했다.

```text
b0d87ad Use Elastic IP for deployment
```

---

## 5. 🔗 무료 임시 도메인 사용

보유한 도메인이 없기 때문에 HTTPS 학습에는 `sslip.io`를 사용했다.

```text
3-34-34-46.sslip.io
        ↓ DNS 조회
3.34.34.46
```

`sslip.io`는 호스트 이름에 포함된 IP 주소를 DNS 응답으로 반환한다. 별도의 도메인 구매나 DNS 레코드 생성 없이 도메인 형태의 주소를 사용할 수 있다.

학습과 테스트에는 편리하지만 외부 서비스에 의존하므로 장기 운영 서비스에는 직접 소유한 도메인을 사용하는 것이 적합하다.

Route 53에서 도메인을 구매할 수도 있지만 도메인 등록료에는 AWS 프로모션 크레딧을 사용할 수 없다. 실제 도메인이 필요해질 때 별도로 구매하기로 했다.

---

## 6. 🔐 Kamal Proxy에서 HTTPS 활성화

`config/deploy.yml`에 다음 설정을 추가했다.

```yaml
proxy:
  ssl: true
  host: 3-34-34-46.sslip.io
```

이 설정은 Kamal 전용 배포 설정이다.

- `host`: 이 애플리케이션으로 전달할 호스트 이름
- `ssl: true`: Kamal Proxy가 Let's Encrypt 인증서를 발급하고 HTTPS를 처리

Kamal Proxy는 EC2의 80번과 443번 포트에서 요청을 받고, 알맞은 Rails 컨테이너로 전달한다. Nginx를 따로 설치하지 않아도 이 역할을 수행한다.

---

## 7. 🚂 Rails에서 HTTPS 인식

`config/environments/production.rb`도 다음과 같이 변경했다.

```ruby
config.assume_ssl = true
config.force_ssl = true
```

- `config.assume_ssl`: 앞단의 프록시가 HTTPS를 처리하고 있다고 Rails가 인식한다.
- `config.force_ssl`: HTTP 요청을 HTTPS로 전환하고 보안 쿠키와 HSTS 관련 동작을 활성화한다.

Kamal Proxy가 외부 TLS 연결을 종료한 뒤 Rails 컨테이너로 요청을 전달하기 때문에 Rails에도 프록시 뒤에서 HTTPS로 서비스 중이라는 사실을 알려줘야 한다.

---

## 8. 🚢 HTTPS 버전 배포

다음 명령으로 변경된 애플리케이션을 배포했다.

```bash
bin/kamal deploy
```

Kamal은 다음 작업을 처리했다.

```text
Rails Docker 이미지 빌드
→ 로컬 Docker Registry에 저장
→ EC2로 이미지 전달
→ 새 Rails 컨테이너 실행
→ Kamal Proxy 설정 적용
→ Let's Encrypt 인증서 발급
→ 새 컨테이너로 트래픽 전환
```

---

## 9. 🧪 HTTPS 확인

HTTPS 상태 확인과 HTTP 리다이렉트를 각각 검사했다.

```bash
curl -I --max-time 15 https://3-34-34-46.sslip.io/up
curl -I --max-time 15 http://3-34-34-46.sslip.io
```

확인한 결과:

- HTTPS `/up` 요청 성공
- 인증서 오류 없음
- HTTP 요청이 HTTPS로 자동 전환됨
- 브라우저에서 앱 화면 정상
- 브라우저의 보안 연결 표시 정상

변경사항은 다음 커밋으로 기록했다.

```text
b31fbec Enable HTTPS deployment
```

---

## 10. 🧭 현재 배포 구조

```text
사용자 브라우저
       │ HTTPS
       ▼
3-34-34-46.sslip.io
       │ DNS
       ▼
Elastic IP 3.34.34.46
       │
       ▼
EC2 보안 그룹: 443 허용
       │
       ▼
Kamal Proxy
TLS 인증서 처리 및 요청 전달
       │
       ▼
Rails Docker 컨테이너
       │ PostgreSQL 5432
       ▼
비공개 RDS PostgreSQL
```

---

## 11. 💡 배운 점

- EC2를 다시 시작하면 자동 공인 IP는 바뀔 수 있다.
- Elastic IP를 사용하면 Kamal과 DNS가 참조할 고정 주소를 만들 수 있다.
- Elastic IP는 EC2를 중지해도 보유 비용이 발생하므로 실습 종료 시 릴리스해야 한다.
- Docker가 다시 실행되면 기존 Rails와 Kamal Proxy 컨테이너도 자동으로 복구될 수 있다.
- Kamal Proxy는 Nginx 없이 외부 요청을 Rails 컨테이너로 전달하고 HTTPS 인증서도 관리할 수 있다.
- HTTPS를 적용하려면 프록시 설정뿐 아니라 Rails가 프록시 뒤의 HTTPS 환경을 인식하도록 설정해야 한다.
- `sslip.io`를 이용하면 도메인을 구매하지 않고도 DNS와 HTTPS 연결 과정을 학습할 수 있다.

---

## 12. ▶️ 다음 작업

Active Storage의 운영 저장소를 EC2 로컬 디스크에서 Amazon S3로 변경한다.

목표:

- 사용자가 업로드한 이미지가 컨테이너 교체나 서버 재배포 후에도 유지된다.
- Rails와 S3 연결 방식을 익힌다.
- S3 접근 권한을 최소 권한 원칙으로 구성한다.
