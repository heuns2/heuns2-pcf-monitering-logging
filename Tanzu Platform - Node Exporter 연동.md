# Tanzu Platform - Node Exporter 연동

- 본 문서는 Runtime Config를 사용하여 Include에 포함 된 Service Tile 또는 모든 VM에 Node Exporter를 설치하여  Health Watch v2.x에서 Linux Node의 Metrics을 연동에 대한 가이드 문서입니다.

## 1. Node Export 설정 준비 

### 1.1. Release upload
- node export release를 bosh director에 업로드 합니다.
- https://github.com/bosh-prometheus/node-exporter-boshrelease/releases

```
$ wget https://github.com/bosh-prometheus/node-exporter-boshrelease/releases/download/v5.0.0/node-exporter-5.0.0.tgz

$ bosh upload-release ./node-exporter-5.0.0.tgz
```

### 1.2. Runtime Config 작성
- Tanzu Platform에 Node Exporter를 공통으로 배치하기 위하여 Runtime Config를 작성 합니다.
- 예시 https://bosh.io/docs/runtime-config/#placement-rules

- 1) appMetrics-03233872d4e216a9b16e, metric-store-3157610394dfaf1e6ce2 Tile만을 적용 할 경우 (Deployment Name 기반)

```
releases:
  - name: node-exporter
    version: 5.0.0

addons:
  - name: node_exporter
    jobs:
      - name: node_exporter
        release: node-exporter
    include:
      deployments:
        - appMetrics-03233872d4e216a9b16e
        - metric-store-3157610394dfaf1e6ce2
    properties: {}
```

- 2) 모든 Tile을 적용 할 경우 (Stemcell OS 기반)

```
releases:
  - name: node-exporter
    version: 5.0.0

addons:
  - name: node_exporter
    jobs:
      - name: node_exporter
        release: node-exporter
    include:
      stemcell:
        - os: ubuntu-xenial
        - os: ubuntu-trusty
    properties: {}
```

### 1.3. Runtime Config Update
- 위에서 작성한 Runtime Config를 Bosh Director에 Update 합니다.

```
$  bosh update-runtime-config node-exporter-runtime-config.yml --name node-exporter
```

- Runtime Config 설정 확인

```
bosh config --name node-exporter --type runtime
Using environment '10.11.45.11' as client 'ops_manager'

ID          30
Type        runtime
Name        node-exporter
Created At  2021-05-13 00:58:11 UTC
Content     ---
            addons:
            - include:
                deployments:
                - appMetrics-03233872d4e216a9b16e
                - metric-store-3157610394dfaf1e6ce2
              jobs:
              - name: node_exporter
                release: node-exporter
              name: node_exporter
              properties: {}
            releases:
            - name: node-exporter
              version: 5.0.0


1 config

Succeeded
```

## 2. 적용 대상의 Tile Apply Change

### 2.1. Tile Apply Change
- 적용 대상의 Deployments를 Apply Change 합니다.
- 만약 초기 Apply Change Log에 아래 사항이 검출이 되지 않으면 적용이 되지 않을 수 있습니다.

```
 addons:
+ - include:
+     deployments:
+     - appMetrics-03233872d4e216a9b16e
+     - metric-store-3157610394dfaf1e6ce2
+   jobs:
+   - name: node_exporter
+     release: node-exporter
+   name: node_exporter
+   properties: {}
```

### 2.2. 적용 확인
- Apply Change가 완료 되면 Node Exporter가 설치 된 VM에 bosh ssh 접속하여 monit summary에 node exporter process가 실행 중임을 확인 합니다.

```
grafana/47aee3d4-14a4-4cb5-995b-a76804ced588:/var/vcap/bosh_ssh/bosh_917d94a13cb5467# monit summary
The Monit daemon 5.2.5 uptime: 1h 8m

Process 'grafana'                   running
Process 'reverse-proxy'             running
Process 'userinfo-proxy'            running
Process 'bosh-dns'                  running
Process 'bosh-dns-resolvconf'       running
Process 'bosh-dns-healthcheck'      running
Process 'system-metrics-agent'      running
Process 'node_exporter'             running
System 'system_localhost'           running
```

## 3.  Health Watch Config 설정

### 3.1. DNS Query
- 내부 DNS 정보를 참고 하기 위하여 Tanzu Platform 내부 아무 VM에 접근하여 아래와 같은 형태로 dig 명령을 실행 합니다.
- dns 참고 링크: https://bosh.io/docs/runtime-config/#placement-rules
- 이때 아래 예시 중 diego-cell, router, uaa는 적용 VM의 instance groups 명 입니다.
- 내부 컴포넌트 dns service discovery 정보는 /var/vcap/instance/dns에 존재합니다.

```
$ dig q-s4.diego-cell.*.*.bosh.
$ dig q-s4.router.*.*.bosh.
$ dig q-s4.uaa.*.*.bosh.

$ dig q-s4.db-and-errand-runner.*.*.bosh.

; <<>> DiG 9.10.3-P4-Ubuntu <<>> q-s4.db-and-errand-runner.*.*.bosh.
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 25552
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;q-s4.db-and-errand-runner.*.*.bosh. IN A

;; ANSWER SECTION:
q-s4.db-and-errand-runner.*.*.bosh. 0 IN A      10.11.45.164

;; Query time: 0 msec
;; SERVER: 169.254.0.2#53(169.254.0.2)
;; WHEN: Thu May 13 02:22:40 UTC 2021
;; MSG SIZE  rcvd: 113

```

### 3.2. Health Watch Scraper 정보 설정


- 아래와 같은 형태의 Scraper를 편집하여 Health Watch v2.x의 [Prometheus Configuration] -> [Additional Scrape Config JobsAdd]에 추가 합니다. 
- 아래 예시는 app-metrics, metrics-store tile을 기준으로 합니다.

```
job_name: app-metrics
metrics_path: /metrics
scheme: http
relabel_configs:
  - source_labels:
      - "__meta_dns_name"
    target_label: scrape_instance_group
    regex: "[^.]*[.]([^.]*).*"
    replacement: "$1"
dns_sd_configs:
  - names:
      - q-s4.db-and-errand-runner.*.*.bosh.
      - q-s4.log-store-vms.*.*.bosh.
    type: A
    port: 9100

job_name: metrics-store
metrics_path: /metrics
scheme: http
relabel_configs:
  - source_labels:
      - "__meta_dns_name"
    target_label: scrape_instance_group
    regex: "[^.]*[.]([^.]*).*"
    replacement: "$1"
dns_sd_configs:
  - names:
      - q-s4.metric-store.*.*.bosh.
    type: A
    port: 9100
```

### 3.2.  Health Watch Apply Change
- Scraper 설정을 끝내고 Health Watch Tile을 Apply Change 합니다.
 

## 4. Grafana 확인

### 4.1. Grafana Dashboard 확인

- Grafana Dashboard에서 node_* 명칭으로 시작하는 Metrics이 수집 되는지 확인 합니다.
