---
title: "AWS EC2의 고정 주소와 Kamal HTTPS 적용하기"
date: 2026-08-24 23:59:00 +0900
categories: [DevOps, AWS]
tags: [Rails, AWS, EC2, RDS, ElasticIP, Docker, Kamal, KamalProxy, HTTPS, TLS, LetsEncrypt, DNS, sslip]
---

> **한 줄 요약**: EC2의 자동 공인 IP는 바뀔 수 있다. Elastic IP로 서버 주소를 고정하고 도메인이 그 주소를 가리키게 한 뒤, Kamal Proxy와 Rails 설정을 함께 맞추면 HTTPS를 적용할 수 있다.

첫 HTTP 배포가 성공했다면 다음 목표는 서버를 다시 시작해도 유지되는 주소와 암호화된 HTTPS 연결이다. 이 글은 EC2와 RDS를 중지했다 다시 시작한 경험에서 출발해, 고정 주소·DNS·TLS 인증서·리버스 프록시의 관계를 하나씩 설명한다.

구체적인 IP와 프로젝트 이름은 환경마다 다르므로 자리표시자로 표현한다. 명령을 실행할 때 `<...>` 부분을 실제 값으로 교체한다.

[이전 글: Rails 앱을 AWS에 배포하기 전 설계하기]({% post_url 2026-08-23-rails-aws-deployment-with-docker-kamal %})

## 1. 중지했던 서비스를 다시 시작한다

### 중지와 종료는 다르다

- **중지(stop)**: 가상 서버 실행을 멈추지만 EBS 디스크와 설정은 남긴다. 다시 시작할 수 있다.
- **종료(terminate)**: EC2 인스턴스를 제거한다. 설정에 따라 디스크도 함께 삭제될 수 있다.
- **재부팅(reboot)**: 같은 인스턴스를 운영체제 수준에서 다시 시작한다. 중지 후 시작과 달리 자동 공인 IP는 일반적으로 유지된다.

비용 절약을 위해 EC2와 RDS를 중지했다면 다음 순서로 복구한다.

1. RDS를 시작한다.
2. 상태가 `Available`이 될 때까지 기다린다.
3. EC2를 시작한다.
4. EC2 상태 검사와 공인 주소를 확인한다.
5. 애플리케이션의 `/up` 경로를 확인한다.

DB가 준비되기 전에 Rails가 시작되면 일시적으로 DB 연결 오류가 날 수 있다. 따라서 RDS를 먼저 시작하고 실제 연결 가능 상태를 확인하는 편이 원인을 구분하기 쉽다.

### 재시작 뒤 컨테이너는 어떻게 되는가?

Docker 서비스와 컨테이너에 적절한 restart 정책이 설정되어 있다면 EC2가 다시 시작될 때 Rails와 Kamal Proxy 컨테이너도 자동으로 실행될 수 있다. 실제로 새 이미지를 다시 배포하지 않았는데도 기존 컨테이너가 복구된 것을 확인할 수 있었다.

다만 “항상 자동 복구된다”고 가정해서는 안 된다. 다음 명령으로 상태를 직접 확인한다.

```bash
bin/kamal server exec 'docker ps'
```

Rails 컨테이너와 `kamal-proxy`가 `Up` 상태라면 실행 중이다. 컨테이너가 없다면 Docker 서비스, restart 정책, Kamal 로그를 차례로 확인한다.

## 2. `/up`으로 애플리케이션을 확인한다

**헬스 체크(health check)**는 애플리케이션이 요청을 받을 준비가 되었는지 확인하는 검사다. Rails는 기본적으로 `/up` 상태 확인 경로를 제공한다.

```bash
curl -I --max-time 15 http://<CURRENT_PUBLIC_IP>/up
```

- `curl`: URL로 HTTP 요청을 보낸다.
- `-I`: 응답 본문 대신 헤더만 확인한다.
- `--max-time 15`: 15초 안에 응답하지 않으면 종료한다.

다음과 같은 성공 상태가 보이면 EC2, Kamal Proxy, Rails 컨테이너까지 요청이 도달한 것이다.

```text
HTTP/1.1 200 OK
```

단, `/up`의 200 응답이 로그인·파일 업로드·백그라운드 작업까지 모두 정상이라는 뜻은 아니다. DB 연결을 검사하도록 헬스 체크를 확장했는지에 따라서도 확인 범위가 달라진다. 배포 직후에는 비민감 화면과 실제 주요 사용자 흐름도 별도로 검사한다.

```text
브라우저 또는 curl
  → EC2 공인 주소
  → Kamal Proxy
  → Rails 컨테이너
  → 필요할 때 RDS PostgreSQL
```

## 3. 자동 공인 IP가 왜 바뀌는가?

EC2가 공개 서브넷에서 자동으로 받은 **public IPv4**는 AWS가 임시로 연결한 주소다. 인스턴스를 중지했다 다시 시작하면 기존 주소를 잃고 새 주소를 받을 수 있다.

실제로 중지했던 EC2를 시작한 뒤 주소가 바뀌어 다음 항목을 다시 수정해야 했다.

- `config/deploy.yml`의 배포 대상
- SSH 접속 주소
- 브라우저의 접속 주소
- 도메인을 사용한다면 DNS 레코드

```yaml
servers:
  web:
    - <CURRENT_PUBLIC_IP>
```

이 경험은 “도메인만 있으면 주소가 안정된다”는 뜻이 아님을 보여 준다. DNS가 가리키는 실제 IP가 바뀌면 DNS 레코드도 고쳐야 한다.

## 4. Elastic IP와 도메인의 역할을 구분한다

### Elastic IP란?

**Elastic IP(EIP)**는 AWS 계정에 할당한 고정 public IPv4 주소다. EC2의 네트워크 인터페이스에 연결하면 인스턴스를 중지했다 시작해도 같은 주소를 다시 사용할 수 있다.

### 도메인과 DNS란?

- **도메인(domain)**: `example.com`처럼 사람이 읽을 수 있는 서비스 이름이다.
- **DNS(Domain Name System)**: 도메인 이름을 실제 IP 주소로 변환하는 시스템이다.
- **A 레코드**: 도메인을 IPv4 주소에 연결하는 DNS 레코드다.

Elastic IP와 도메인은 대체재가 아니라 서로 다른 문제를 해결한다.

```text
example.com           사람이 기억하는 이름
     │ DNS A 레코드
     ▼
<ELASTIC_IP>          변하지 않는 서버 주소
     │
     ▼
EC2
```

단일 EC2 서비스에서는 보통 다음 순서로 구성한다.

1. Elastic IP를 할당한다.
2. EC2에 연결한다.
3. Kamal 배포 대상과 SSH 주소를 Elastic IP로 바꾼다.
4. 도메인의 A 레코드가 Elastic IP를 가리키게 한다.

```yaml
servers:
  web:
    - <ELASTIC_IP>
```

Elastic IP가 연결되면 기존 자동 공인 IP는 더 이상 사용하지 않는다. AWS는 현재 Elastic IP뿐 아니라 EC2에 자동 할당된 public IPv4에도 요금을 부과하므로 “고정 IP만 유료”라고 이해하면 안 된다. 사용하지 않는 Elastic IP는 연결 해제만 하지 말고 계정에서 릴리스해야 한다.

여러 EC2로 확장하거나 서버 한 대의 장애를 자동으로 우회해야 한다면 Elastic IP를 각 서버에 직접 연결하는 대신 ALB와 Route 53 같은 구조를 검토한다.

## 5. 도메인이 올바른 서버를 가리키는지 확인한다

운영 서비스에는 직접 소유한 도메인을 사용하는 것이 일반적이다.

```text
example.com      A → <ELASTIC_IP>
www.example.com  A → <ELASTIC_IP>  # 이 주소도 사용할 때만
```

DNS의 **TTL(Time To Live)**은 다른 DNS 서버나 컴퓨터가 조회 결과를 얼마 동안 캐시할지 나타낸다. 레코드를 바꾼 직후에는 이전 값이 TTL 동안 남아 있을 수 있다.

```bash
dig +short example.com
```

출력이 `<ELASTIC_IP>`와 같으면 기본 DNS 연결이 된 것이다. A 레코드뿐 아니라 AAAA 레코드도 확인한다. 잘못된 IPv6 주소가 남아 있으면 일부 사용자는 다른 서버로 연결될 수 있다.

### 도메인이 없을 때 학습하는 방법

실습 당시에는 별도 도메인을 구매하지 않고 `sslip.io` 같은 IP 기반 와일드카드 DNS 서비스를 사용했다.

```text
<IP를 포함한 호스트 이름>.sslip.io
  → DNS 조회
<해당 IP>
```

별도의 DNS 레코드 없이 호스트 이름을 만들 수 있어 학습에는 편리하다. 하지만 외부 서비스의 정책과 가용성에 의존하고 주소가 서비스 이름에 노출되므로 장기 운영에는 직접 소유한 도메인이 적합하다.

2026년부터 Let's Encrypt는 짧은 수명의 IP 주소 인증서도 지원한다. 그렇더라도 일반적인 Kamal Proxy 자동 HTTPS 구성은 `host`에 도메인 이름을 지정하는 흐름이 가장 단순하다. IP 인증서를 사용하려면 현재 Kamal Proxy와 ACME 클라이언트가 이를 지원하는지 별도로 확인해야 한다.

## 6. HTTPS와 TLS 인증서의 역할

**HTTP**는 요청과 응답을 암호화하지 않는다. **HTTPS**는 HTTP 통신을 TLS(Transport Layer Security)로 암호화한다.

TLS 인증서는 다음을 돕는다.

- 사용자가 의도한 서버에 연결했는지 확인
- 로그인 정보와 쿠키 등 전송 내용을 암호화
- 전송 중 데이터가 변조되었는지 탐지

**Let's Encrypt**는 도메인이나 IP 주소의 통제권을 자동으로 검증하고 무료 TLS 인증서를 발급하는 인증 기관이다. 인증서에는 만료 기간이 있으므로 자동 갱신이 중요하다.

## 7. Kamal Proxy의 자동 HTTPS 조건을 준비한다

Kamal Proxy는 EC2의 80번과 443번 포트에서 요청을 받고, Rails 컨테이너로 전달하는 리버스 프록시다. `ssl: true`를 사용하면 Let's Encrypt 인증서의 발급과 갱신도 처리한다.

Kamal 공식 문서를 기준으로 자동 HTTPS에는 다음 조건이 필요하다.

- 한 대의 서버에 배포한다.
- `proxy.host`가 설정되어 있다.
- 해당 host가 실제 배포 서버를 가리킨다.
- 외부에서 서버의 443번 포트에 접근할 수 있다.
- 다른 Nginx, Caddy, Apache 등이 80·443번 포트를 이미 점유하지 않는다.

80번 포트는 Kamal Proxy의 기본 HTTP→HTTPS 리다이렉트를 위해 함께 연다. Kamal Proxy의 자동 인증서 검증을 HTTP-01 방식이라고 단정하고 80번만 점검하면 원인을 잘못 찾을 수 있다. 공식 안내에서는 443번 접근 가능 여부가 핵심 조건이다.

EC2 보안 그룹은 다음처럼 구성한다.

| 포트 | 용도 | 출발지 |
| :--- | :--- | :--- |
| TCP 80 | HTTP 요청과 HTTPS 리다이렉트 | `0.0.0.0/0` |
| TCP 443 | HTTPS와 인증서 검증 | `0.0.0.0/0` |

IPv6를 서비스한다면 해당 IPv6 규칙도 추가한다.

## 8. Kamal 설정에서 HTTPS를 활성화한다

`config/deploy.yml`에 서비스가 사용할 호스트 이름을 지정한다.

```yaml
proxy:
  ssl: true
  host: example.com
```

- `host`: 이 Rails 앱으로 전달할 HTTP Host 이름이다. DNS가 배포 서버를 가리켜야 한다.
- `ssl: true`: Kamal Proxy가 자동 TLS를 사용하도록 한다.

Kamal Proxy는 Rails 컨테이너 밖에서 동작한다.

```text
브라우저
  │ HTTPS 443
  ▼
Kamal Proxy
  │ TLS 종료 후 Docker 내부 통신
  ▼
Rails 컨테이너
```

**TLS 종료(termination)**란 암호화된 HTTPS 연결을 프록시가 풀고 내부 애플리케이션에는 일반 HTTP 요청으로 전달하는 구조를 뜻한다. 서버 내부 통신 구조와 외부 통신 구조가 다르기 때문에 Rails에도 이 사실을 알려야 한다.

## 9. Rails가 프록시 뒤의 HTTPS를 인식하게 한다

`config/environments/production.rb`에서 다음 설정을 확인한다.

```ruby
config.assume_ssl = true
config.force_ssl = true
```

- `config.assume_ssl`: 프록시가 외부 HTTPS를 처리했다고 Rails가 가정하게 한다.
- `config.force_ssl`: HTTP 요청을 HTTPS로 리다이렉트하고 secure cookie와 HSTS 같은 보안 동작을 활성화한다.

Kamal Proxy에서 `ssl: true`를 사용하면 기본적으로 전달 헤더 동작도 달라질 수 있다. 다른 CDN이나 로드 밸런서를 추가해 TLS를 종료한다면 `X-Forwarded-Proto`, `forward_headers`, 신뢰할 프록시 범위를 함께 검토해야 한다. 신뢰하지 않는 인터넷 요청의 전달 헤더를 그대로 믿도록 설정하면 안 된다.

## 10. 배포한 뒤 단계별로 검증한다

설정을 반영한다.

```bash
bin/kamal deploy
```

Kamal은 대략 다음 순서로 작업한다.

```text
Docker 이미지 빌드
  → 레지스트리에 저장
  → EC2로 이미지 전달
  → 새 Rails 컨테이너 실행
  → 헬스 체크
  → Proxy와 TLS 설정 적용
  → 새 컨테이너로 트래픽 전환
```

### 1단계: HTTPS 상태 확인

```bash
curl -I --max-time 15 https://example.com/up
```

확인할 내용:

- 성공 응답인가?
- 인증서 오류가 없는가?
- 인증서의 호스트 이름이 접속 주소와 일치하는가?

### 2단계: HTTP 리다이렉트 확인

```bash
curl -I --max-time 15 http://example.com
```

`Location: https://example.com/...`과 3xx 상태 코드가 보이면 HTTPS로 이동하고 있는 것이다.

### 3단계: 실제 사용자 흐름 확인

- 브라우저에서 보안 연결 표시 확인
- 로그인과 로그아웃
- form 제출과 CSRF 동작
- 세션 쿠키 유지
- 이미지와 정적 파일 로드

이 단계가 필요한 이유는 `/up`이 성공해도 쿠키, 프록시 헤더, URL 생성 설정은 별도로 잘못될 수 있기 때문이다.

## 11. 문제가 생겼을 때 한 단계씩 좁힌다

| 증상 | 먼저 확인할 곳 |
| :--- | :--- |
| 도메인이 다른 IP를 반환 | DNS A/AAAA 레코드와 TTL |
| 연결 시간 초과 | EC2 상태, Elastic IP 연결, 보안 그룹, 라우팅 |
| 인증서 발급 실패 | host가 서버를 가리키는지, 443번 접근, 포트 점유 |
| 502/503 응답 | Rails 컨테이너 상태, `/up`, Kamal Proxy 로그 |
| HTTP가 HTTPS로 가지 않음 | `proxy.ssl`, 기본 `ssl_redirect`, 80번 접근 |
| 반복 리다이렉트 | `assume_ssl`, TLS 종료 위치, 전달 헤더 |
| 로그인 세션이 유지되지 않음 | secure cookie, host, Rails HTTPS 인식 |

로그와 컨테이너 상태를 확인한다.

```bash
bin/kamal app logs
bin/kamal server exec 'docker ps'
```

명령 이름과 옵션은 Kamal 버전에 따라 달라질 수 있으므로 다음 명령으로 현재 프로젝트의 사용법을 확인한다.

```bash
bin/kamal help
```

한 번에 여러 설정을 바꾸기보다 DNS → 네트워크 → Proxy → Rails 순서로 확인하면 실패 원인을 찾기 쉽다.

## 12. 여기까지 완료한 뒤 남는 운영 과제

HTTPS가 적용되었다고 배포가 끝난 것은 아니다.

- Active Storage 업로드를 EC2 로컬 디스크 대신 S3에 저장
- RDS 자동 백업과 실제 복구 연습
- 삭제 방지와 스냅샷 보존 정책 확인
- 애플리케이션·프록시 로그와 오류 추적
- OS, Docker, Rails 의존성 보안 업데이트
- 비밀값 교체와 IAM 권한 정기 점검
- 여러 EC2로 확장할 때 ALB와 공유 저장소 도입

특히 컨테이너가 교체되거나 EC2가 사라져도 남아야 하는 데이터는 서버 로컬 파일 시스템에만 두지 않는다. 데이터베이스는 RDS에, 사용자 업로드는 S3 같은 외부 저장소에 두는 것이 다음 단계다.

## 참고 문서

- [AWS: EC2 중지와 시작 시 변경되는 항목](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/how-ec2-instance-stop-start-works.html)
- [AWS: EC2 public IPv4 주소](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-instance-addressing.html)
- [Kamal: Proxy 설정](https://kamal-deploy.org/docs/configuration/proxy/)
- [Rails: `config.assume_ssl` 설정](https://guides.rubyonrails.org/configuring.html#config-assume-ssl)
- [Let's Encrypt: IP 주소 인증서 정식 지원](https://letsencrypt.org/2026/01/15/6day-and-ip-general-availability.html)
