# Prometheus target 환경 분리

> 카테고리: 모니터링 · 2026-06 · 도메인: Prometheus · Grafana · 멀티 인스턴스

## 문제

Prometheus 설정을 멀티 인스턴스 기준으로 바꾸는 과정에서 로컬 실행 방식과 운영 실행 방식이 서로 달라 혼란이 생겼다.

로컬은 두 가지 방식으로 실행된다.

| 로컬 실행 방식 | 앱 컨테이너 |
| --- | --- |
| 싱글 인스턴스 | `for-pets-app` |
| 멀티 인스턴스 | `for-pets-app-blue`, `for-pets-app-green` |

반면 운영 EC2는 아래 이름을 사용한다.

```text
forpets-blue
forpets-green
```

처음에는 하나의 `monitoring/prometheus.yml`에 운영 target을 넣었다.

```yaml
targets:
  - 'forpets-blue:8080'
  - 'forpets-green:8080'
```

이 설정은 운영에서는 맞지만, 로컬 싱글 인스턴스에서는 `forpets-blue`, `forpets-green`이라는 컨테이너가 없어 scrape에 실패한다.

## 원인

Prometheus가 Docker 컨테이너로 실행될 때 target은 **호스트 포트 기준이 아니라 Docker 네트워크 내부 이름 기준**이다.

예를 들어 로컬 멀티에서는 host port가 아래처럼 열려 있다.

```text
for-pets-app-blue  : 8080:8080
for-pets-app-green : 8081:8080
```

하지만 Prometheus 컨테이너가 같은 Docker 네트워크 안에서 접근할 때는 둘 다 내부 포트 `8080`으로 접근한다.

```text
for-pets-app-blue:8080
for-pets-app-green:8080
```

운영도 마찬가지다.

```text
forpets-blue:8080
forpets-green:8080
```

환경마다 컨테이너 이름이 다르기 때문에 하나의 Prometheus 설정 파일로 모두 커버하려 하면 특정 환경에서 깨진다.

## 해결

Prometheus 설정 파일을 환경별로 분리했다.

```text
monitoring/
├── prometheus-single.yml
├── prometheus-multi-local.yml
└── prometheus-prod.yml
```

각 파일의 target은 다음과 같다.

| 설정 파일 | 대상 환경 | target |
| --- | --- | --- |
| `prometheus-single.yml` | 로컬 싱글 | `for-pets-app:8080` |
| `prometheus-multi-local.yml` | 로컬 멀티 | `for-pets-app-blue:8080`, `for-pets-app-green:8080` |
| `prometheus-prod.yml` | 운영 멀티 | `forpets-blue:8080`, `forpets-green:8080` |

로컬 Docker Compose에서는 `PROMETHEUS_CONFIG` 환경변수로 어떤 설정 파일을 마운트할지 선택하게 했다.

```yaml
volumes:
  - ${PROMETHEUS_CONFIG:-./monitoring/prometheus-single.yml}:/etc/prometheus/prometheus.yml
```

로컬 멀티 실행은 Makefile로 감쌌다.

```bash
make multi-up
```

내부적으로는 아래 명령이 실행된다.

```bash
PROMETHEUS_CONFIG=./monitoring/prometheus-multi-local.yml \
docker compose --profile multi up -d mysql redis kafka qdrant app-blue app-green nginx-multi prometheus grafana
```

## 결과

로컬 싱글, 로컬 멀티, 운영 멀티 모두 각 환경에 맞는 Prometheus target을 사용하게 됐다.

운영 배포 후 Prometheus API로 확인한 결과 두 target 모두 `UP` 상태였다.

```text
forpets-blue:8080   up
forpets-green:8080  up
```

## 배운 점

Prometheus target은 "내 PC에서 보이는 포트"가 아니라 **Prometheus 프로세스가 바라보는 네트워크 기준**으로 작성해야 한다.

| Prometheus 위치 | target 기준 |
| --- | --- |
| Docker 컨테이너 | Docker service/container DNS |
| EC2에 직접 설치 | `localhost:포트` |
| 외부 PC | public IP 또는 domain |

또한 `job_name`은 단순 이름이 아니라 Prometheus label로 남는다.

```promql
up{job="forpets-prod-multi"}
```

Grafana 쿼리에서도 사용될 수 있으므로 환경이 드러나는 안정적인 이름으로 고정하는 것이 좋다.
