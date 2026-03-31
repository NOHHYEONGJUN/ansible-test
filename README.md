# AWX Playbook: OTel Collector Deployment

이 폴더는 AWX Project로 연결해 실제 VM에 OpenTelemetry Collector를 배포하기 위한 파일입니다.

## 파일

- `deploy-otel-collector.yml`: 설치/설정/서비스 구동 플레이북
- `deploy-otel-collector-with-loki.yml`: 위와 동일 계열 + **로그(filelog) → Loki OTLP** (`awx-example/loki-playbook.md` 참고)
- `remove-otel-collector.yml`: 위 배포본 제거(서비스 중지·유닛·`/opt`·`/etc/otelcol-contrib`·선택 시 `otelcol` 사용자). 재설치 테스트용
- `templates/config.yaml.j2`: Collector 설정 템플릿(메트릭 중심)
- `templates/config-with-loki.yaml.j2`: 메트릭 + Loki 로그용 설정 템플릿
- `templates/otelcol-contrib.service.j2`: systemd 유닛 템플릿

## AWX에서 사용 방법

1. AWX Project의 소스 루트를 이 저장소로 지정
2. Job Template에서 Playbook을 `awx-playbook/deploy-otel-collector.yml`(설치), `awx-playbook/deploy-otel-collector-with-loki.yml`(메트릭+Loki 로그), 또는 `awx-playbook/remove-otel-collector.yml`(제거)로 선택
3. 실행할 호스트가 들어 있는 **인벤토리**를 지정해 실행

제거 플레이북은 **`otel_prom_rw_endpoint`가 필요 없습니다.** Extra vars로 `otel_remove_service_user: false` 를 주면 `otelcol` 사용자만 남깁니다.

### 한 개 템플릿으로 여러 인벤토리 쓰기 (실행 시 프롬프트)

Job Template에서 다음을 켜두면, 실행할 때마다 인벤토리를 고를 수 있습니다.

| 옵션 | 권장 |
|------|------|
| **Prompt on launch: Inventory** | ✅ 켜기 (`ask_inventory_on_launch`) |
| **Prompt on launch: Limit** | 호스트만 골라 돌릴 때 켜기 (`ask_limit_on_launch`) |
| **Prompt on launch: Credentials** | 실행 시 SSH Machine Credential을 고를 때 켜기 |

플레이북은 `hosts: all` 이므로, **선택한 인벤토리에 포함된 호스트 전체**가 대상입니다. 특정 VM만 돌리려면 Limit에 호스트 이름을 입력하세요.

> O’watch 웹 **자동화 배포** API는 `launch` JSON에 `inventory`·`limit`·`credentials`를 넘기므로, 위 프롬프트가 켜져 있어야 API 연동이 동작합니다. API 서버(`server/index.mjs`)는 기동 시 해당 템플릿에 프롬프트를 자동으로 켜려고 시도합니다(`.env`에서 `AWX_AUTO_PATCH_TEMPLATE=false` 로 끌 수 있음).

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

### Loki 로그 포함 플레이북 extra_vars 예시

`deploy-otel-collector-with-loki.yml` 사용 시 **`otel_loki_endpoint`**(Loki OTLP HTTP 베이스, 예: `http://loki:3100`)가 필요합니다. 템플릿이 자동으로 `/otlp` 경로를 붙입니다.

```yaml
otel_prom_rw_endpoint: "http://<mimir-or-gateway>:8080/api/v1/push"
otel_prom_rw_org_id: "anonymous"
otel_loki_endpoint: "http://<loki-host>:3100"
monitor_metrics: true
monitor_logs: true
```

## 배포 후 확인

- 서비스 상태: `systemctl status otelcol-contrib`
- 포트 확인: `ss -lntp | egrep ':4317|:8888'`
- Collector 로그: `journalctl -u otelcol-contrib -f`

## 참고

- `host.name`은 `inventory_hostname`으로 자동 설정됩니다.
- 메트릭만 쓰면 `config.yaml.j2` / `deploy-otel-collector.yml` 을 사용합니다.
- Loki까지 쓰면 `config-with-loki.yaml.j2` / `deploy-otel-collector-with-loki.yml` 을 사용합니다(로그 수집 시 `otelcol` 사용자를 `adm` 그룹에 추가).
