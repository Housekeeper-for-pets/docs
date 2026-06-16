# 단일 EC2 내부 멀티 인스턴스 운영 전환

> 카테고리: 배포/인프라 · 2026-06 · 도메인: CI/CD · Docker Compose · Nginx

## 문제

초기 운영 배포는 GitHub Actions에서 EC2에 SSH 접속한 뒤 `docker run` 명령으로 컨테이너를 직접 실행하는 방식이었다.

처음에는 Spring Boot 앱 1개만 띄우면 됐지만, 이후 운영 구조가 아래처럼 바뀌었다.

```text
EC2 1대
├── Nginx
├── forpets-blue
├── forpets-green
├── Qdrant
├── Prometheus
└── Grafana
```

컨테이너가 늘어나자 `ci.yml` 안에 긴 `docker run` 명령을 계속 추가해야 했다.  
이 방식은 다음 문제가 있었다.

| 문제 | 설명 |
| --- | --- |
| 배포 스크립트가 길어짐 | 컨테이너 옵션, 환경변수, 볼륨, 네트워크가 모두 `ci.yml`에 섞임 |
| 운영 구조 파악이 어려움 | 실제 어떤 컨테이너가 떠야 하는지 파일 하나로 보기 어려움 |
| Nginx 502 발생 | Nginx는 떴지만 앱 컨테이너가 없거나 이름이 맞지 않으면 upstream 연결 실패 |
| 모니터링 추가가 어려움 | Prometheus/Grafana까지 `docker run`으로 관리하면 유지보수 부담 증가 |

## 원인

`ci.yml`은 배포 자동화 파일이지, 운영 컨테이너 구성을 표현하기 좋은 파일이 아니다.

기존 구조에서는 GitHub Actions가 아래 책임을 모두 가지고 있었다.

```text
1. 이미지 pull
2. 네트워크 생성
3. Nginx conf 생성
4. 앱 컨테이너 2개 실행
5. Nginx 실행
6. health check
```

운영 컨테이너 구성이 복잡해질수록 CI 스크립트가 인프라 정의 파일처럼 변해버렸다.

## 해결

운영용 Compose 파일을 별도로 만들었다.

```text
docker-compose.prod.yml
nginx/prod.conf
monitoring/prometheus-prod.yml
```

운영 Compose에는 실제 운영에 필요한 컨테이너만 포함했다.

| 컨테이너 | 역할 |
| --- | --- |
| `forpets-nginx` | 외부 요청 진입점, blue/green upstream |
| `forpets-blue` | Spring Boot 앱 인스턴스 1 |
| `forpets-green` | Spring Boot 앱 인스턴스 2 |
| `forpets-qdrant` | 벡터 검색 |
| `forpets-prometheus` | 메트릭 수집 |
| `forpets-grafana` | 메트릭 시각화 |

반대로 운영에서 AWS 관리형 서비스를 쓰는 것은 Compose에서 제외했다.

| 리소스 | 운영 방식 |
| --- | --- |
| MySQL | RDS |
| Redis | ElastiCache |

GitHub Actions 역할도 단순화했다.

```text
GitHub Actions
→ ECR 이미지 push
→ EC2에 compose/nginx/monitoring 파일 업로드
→ EC2에서 .env.prod 생성
→ docker compose -f docker-compose.prod.yml up -d
→ health check
```

## 결과

운영 컨테이너 구성을 파일 하나로 볼 수 있게 됐다.

```bash
docker compose --env-file .env.prod -f docker-compose.prod.yml ps
```

운영 확인 결과는 아래와 같았다.

| 항목 | 결과 |
| --- | --- |
| `forpets-blue:8080/health` | OK |
| `forpets-green:8081/health` | OK |
| `forpets-nginx:80/health` | OK |
| Prometheus target | blue/green 모두 UP |
| Qdrant | `/readyz`, `/healthz` 정상 |

## 주의점

현재 구조는 이름이 blue/green이지만 엄밀한 Blue-Green Deployment는 아니다.

```text
현재 구조:
blue와 green이 동시에 운영 트래픽을 받는 멀티 인스턴스

진짜 Blue-Green:
blue는 현재 운영 버전
green은 새 버전
검증 후 Nginx 또는 LB 트래픽을 green으로 전환
실패 시 blue로 rollback
```

따라서 현재 구조는 아래처럼 표현하는 것이 정확하다.

```text
단일 EC2 내부 멀티 인스턴스 + Docker Compose 운영 배포
```

## 배운 점

CI 파일과 운영 Compose 파일의 책임은 다르다.

| 파일 | 책임 |
| --- | --- |
| `ci.yml` | 언제, 어떤 순서로 배포할지 |
| `docker-compose.prod.yml` | 운영에서 어떤 컨테이너를 어떤 설정으로 띄울지 |

컨테이너가 1개일 때는 `docker run`도 충분하지만, Nginx, 앱 2개, Qdrant, Prometheus, Grafana까지 늘어나면 Compose로 운영 구성을 명시하는 편이 훨씬 안전하다.
