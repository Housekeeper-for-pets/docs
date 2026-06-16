# EC2 Docker Compose 플러그인 누락으로 배포 실패

> 카테고리: 배포/CI · 2026-06 · 도메인: GitHub Actions · EC2

## 문제

운영 배포 방식을 `docker run`에서 `docker-compose.prod.yml` 기반으로 변경한 뒤 GitHub Actions 배포가 실패했다.

에러 메시지는 아래와 같았다.

```text
❌ EC2에 Docker Compose 플러그인이 없습니다. docker compose version이 성공해야 합니다.
Process exited with status 1
```

Docker 이미지는 정상적으로 ECR에서 pull 됐지만, EC2에서 `docker compose version` 명령이 실패하면서 배포가 중단됐다.

## 원인

EC2에는 Docker Engine만 설치되어 있었고, Docker Compose v2 플러그인은 설치되어 있지 않았다.

헷갈렸던 지점은 아래 차이다.

| 명령 | 의미 |
| --- | --- |
| `docker ps` | Docker Engine이 있으면 동작 |
| `docker compose version` | Docker Compose v2 플러그인이 있어야 동작 |

기존 배포는 `docker run`만 사용했기 때문에 Compose 플러그인이 없어도 문제가 없었다.  
하지만 운영 배포를 Compose 기반으로 바꾸면서 EC2에 Compose 플러그인이 필요해졌다.

## 해결

CI 배포 스크립트에서 Compose 플러그인이 없으면 자동 설치하도록 처리했다.

```bash
if ! docker compose version >/dev/null 2>&1; then
  echo "EC2에 Docker Compose 플러그인이 없어 설치를 시도합니다."
  sudo mkdir -p /usr/local/lib/docker/cli-plugins
  sudo curl -SL "https://github.com/docker/compose/releases/download/v2.29.7/docker-compose-linux-x86_64" \
    -o /usr/local/lib/docker/cli-plugins/docker-compose
  sudo chmod +x /usr/local/lib/docker/cli-plugins/docker-compose
fi

if ! docker compose version >/dev/null 2>&1; then
  echo "Docker Compose 플러그인 설치 후에도 docker compose version이 실패했습니다."
  exit 1
fi
```

설치 위치는 Docker CLI 플러그인 표준 경로를 사용했다.

```text
/usr/local/lib/docker/cli-plugins/docker-compose
```

## 결과

다음 배포부터 EC2에 Compose 플러그인이 없으면 CI가 자동 설치한 뒤 배포를 이어갈 수 있게 됐다.

배포 흐름은 아래처럼 정리됐다.

```text
ECR 로그인
→ 최신 이미지 pull
→ docker compose version 확인
→ 없으면 compose plugin 설치
→ docker compose -f docker-compose.prod.yml up -d
→ health check
```

## 배운 점

배포 방식을 바꾸면 애플리케이션 코드뿐 아니라 **서버에 설치되어 있어야 하는 런타임 도구**도 함께 확인해야 한다.

이번 경우에는 Docker 자체가 아니라 Docker Compose 플러그인이 빠져 있었다.  
`docker ps`가 된다고 해서 `docker compose`도 당연히 된다고 생각하면 안 된다.

면접 포인트로는 Docker Compose v1과 v2의 차이를 설명할 수 있으면 좋다.

| 버전 | 명령 |
| --- | --- |
| Compose v1 | `docker-compose` |
| Compose v2 | `docker compose` |

현재는 Docker CLI 플러그인 방식인 v2가 일반적이다.
