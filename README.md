# AWX Playbook: OTel Collector Deployment

이 폴더는 AWX Project로 연결해 실제 VM에 OpenTelemetry Collector를 배포하기 위한 파일입니다.

## 파일

- `deploy-otel-collector.yml`: 설치/설정/서비스 구동 플레이북
- `templates/config.yaml.j2`: Collector 설정 템플릿
- `templates/otelcol-contrib.service.j2`: systemd 유닛 템플릿

## AWX에서 사용 방법

1. AWX Project의 소스 루트를 이 저장소로 지정
2. Job Template에서 Playbook을 `awx-playbook/deploy-otel-collector.yml`로 선택
3. Inventory/Host를 선택해 실행

## 필수 / 권장 변수

퍼블릭 GitHub에는 내부 URL을 두지 않습니다. **`otel_prom_rw_endpoint`는 반드시 AWX에서 지정**해야 합니다(비어 있으면 플레이북이 시작 시 실패합니다).

## Launch 시 extra_vars 예시

```yaml
otel_prom_rw_endpoint: "http://<mimir-or-gateway>:8080/api/v1/push"
otel_prom_rw_org_id: "anonymous"
# 선택: 배포 요약에 Grafana 한 줄 출력
# owatch_grafana_url: "https://<grafana-host>:3000"

otel_version: "0.102.1"
monitor_metrics: true
monitor_logs: false
monitor_traces: false
```

## 배포 후 확인

- 서비스 상태: `systemctl status otelcol-contrib`
- 포트 확인: `ss -lntp | egrep ':4317|:8888'`
- Collector 로그: `journalctl -u otelcol-contrib -f`

## 참고

- `host.name`은 `inventory_hostname`으로 자동 설정됩니다.
- 현재 템플릿은 메트릭 파이프라인 중심이며, 로그/트레이스 파이프라인은 추후 확장 가능합니다.
