# Grafana 대시보드가 배포에 없거나 No data — 프로비저닝과 datasource UID

> 카테고리: 모니터링 · 2026-06-10(배포검증) · 도메인: Grafana

## 문제
로컬에서 만든 대시보드가 배포 Grafana에 안 보이고, export JSON을 올려도 배포에서 **No data**.

## 원인
- 로컬/배포 Grafana는 **별개 서버·DB**라 계정이 같아도 대시보드가 공유되지 않는다. 팀 대시보드는 **레포 JSON 프로비저닝(dashboard-as-code)**으로 관리해야 배포에 자동 등장한다.
- "Export for sharing externally"의 `${DS_PROMETHEUS}` + `__inputs`는 **UI Import 전용**이다. 파일 프로비저닝은 이 변수를 풀어주지 않아 datasource가 미해결 → **No data**.

## 해결
**클래식 포맷**으로 export → `__inputs` 제거 + datasource uid를 **배포 Prometheus의 uid로 하드코딩** → `provisioning/dashboards/` 폴더에 커밋.

## 배운 점
대시보드는 처음부터 **레포 provisioning으로 관리**한다. No data의 1순위 원인은 **datasource UID 불일치**이므로, 배포 Grafana의 Prometheus UID와 JSON의 UID가 같은지부터 확인한다.
