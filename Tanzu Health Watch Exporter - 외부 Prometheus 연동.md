# 1. Tanzu Health Watch Exporter - 외부 Prometheus 연동

- 본 가이드는 Health Watch Exporter를 바탕으로 외부에 존재하는 Prometheus에 연동하고, 해당 Prometheus를 Grafana에 연동하여 사용하는 방안에 대해서 설명합니다.
- Health Watch v2.x는 Grafana - Prometheus로 아키텍처가 변경 되었고 Exporter Tile을 기반으로 모든 Platform의 Metrics을 수집하게 됩니다.
- Tanzu Platform의 모든 VM 아래의 job을 통하여 prom_scraper 기능이 활성화 되어 있습니다, 아래의 정보를 바탕으로 내부 Bosh DNS를 기반으로 Health Watch Exporter가 모든 VM의 Metrics을 수집 합니다. 

```
$ monit summary
The Monit daemon 5.2.5 uptime: 2d 2h 18m

Process 'gorouter'                  running
Process 'loggregator_agent'         running
Process 'loggr-syslog-agent'        running
Process 'metrics-discovery-registrar' running
Process 'metrics-agent'             running
Process 'loggr-forwarder-agent'     running
Process 'loggr-udp-forwarder'       running
Process 'prom_scraper'              running
Process 'bosh-dns'                  running
Process 'bosh-dns-resolvconf'       running
Process 'bosh-dns-healthcheck'      running
Process 'system-metrics-agent'      running
System 'system_localhost'           running

$ cat prom_scraper_config.yml
---
port: 14821
source_id: "prom_scraper"
instance_id: 708eb0af-7e9c-4f29-a74c-de0497d76c14
scheme: https
server_name: prom_scraper_metricsrouter/708eb0af-7e9c-4f29-a74c-


$ cat bpm.yml
---
processes:
- name: prom_scraper
  executable: "/var/vcap/packages/prom_scraper/prom-scraper"
  unsafe:
    unrestricted_volumes:
    - path: "/var/vcap/jobs/*/config/prom_scraper_config.yml"
      mount_only: true
    - path: "/var/vcap/jobs/*/config/metric_port.yml"
      mount_only: true
  env:
    LOGGREGATOR_AGENT_ADDR: localhost:3458
    CA_CERT_PATH: "/var/vcap/jobs/prom_scraper/config/certs/loggregator_ca.crt"
    CLIENT_CERT_PATH: "/var/vcap/jobs/prom_scraper/config/certs/loggregator_agent.crt"
    CLIENT_KEY_PATH: "/var/vcap/jobs/prom_scraper/config/certs/loggregator_agent.key"
    CONFIG_GLOBS: "/var/vcap/jobs/*/config/prom_scraper_config.yml,/var/vcap/jobs/*/config/metric_port.yml"
    SCRAPE_INTERVAL: 60s
    DEFAULT_SOURCE_ID: router
    SKIP_SSL_VALIDATION: 'false'
    METRICS_PORT: '14821'
    METRICS_CA_FILE_PATH: "/var/vcap/jobs/prom_scraper/config/certs/metrics_ca.crt"
    METRICS_CERT_FILE_PATH: "/var/vcap/jobs/prom_scraper/config/certs/metrics.crt"
    METRICS_KEY_FILE_PATH: "/var/vcap/jobs/prom_scraper/config/certs/metrics.key"
    SCRAPE_CA_CERT_PATH: "/var/vcap/jobs/prom_scraper/config/certs/scrape_ca.crt"
    SCRAPE_CERT_PATH: "/var/vcap/jobs/prom_scraper/config/certs/scrape.crt"
    SCRAPE_KEY_PATH: "/var/vcap/jobs/prom_scraper/config/certs/scrape.key"
```

## 1.1. Grafana - Prometheus Install

```
# prometheus install
$ sudo apt-get install -y prometheus prometheus-pushgateway prometheus-alertmanager


# grafana install
$ sudo add-apt-repository "deb https://packages.grafana.com/oss/deb stable main"

$ curl https://packages.grafana.com/gpg.key | sudo apt-key add -
$ sudo apt-get install grafana

# grafana server 실행
$ sudo systemctl daemon-reload
$ sudo systemctl start grafana-server
$ sudo systemctl enable grafana-server
```

## 2.1. Health Watch Export 연동

### 2.1.1. Health Watch v2.x TSDB VM 내부 작업
- Health Watch VM의 tsdb VM 내부에 접속하여 Exporter 연동에 사용 될 정보를 확인 합니다.

```
# Health Watch Deployments에 tsdb VM 접속
$ bosh -d p-healthwatch2-d7254edfb0ac9ee0208e ssh tsdb/0 

# VM 내부에서 아래 디렉토리로 이동
$ cd /var/vcap/jobs/prometheus/config/certs

# 아래 3종의 CA, 인증서를 Prometheus 실행 VM으로 복사 또는 이동
$ ls –al
-rw-r----- 1 root vcap 1208 Apr  7 04:03 alertmanager_ca.pem
-rw-r----- 1 root vcap 1680 Apr  7 04:03 alertmanager_certificate.key
-rw-r----- 1 root vcap 1250 Apr  7 04:03 alertmanager_certificate.pem
-rw-r----- 1 root vcap    1 Apr  7 04:03 grafana_ca.pem
-rw-r----- 1 root vcap 1209 Apr  7 04:03 healthwatch_exporter_ca.pem
-rw-r----- 1 root vcap 1676 Apr  7 04:03 healthwatch_exporter_certificate.key
-rw-r----- 1 root vcap 1250 Apr  7 04:03 healthwatch_exporter_certificate.pem
-rw-r----- 1 root vcap 1209 Apr  7 04:03 pks_cluster_discovery_ca.pem
-rw-r----- 1 root vcap 1680 Apr  7 04:03 pks_cluster_discovery_certificate.key
-rw-r----- 1 root vcap 1246 Apr  7 04:03 pks_cluster_discovery_certificate.pem
-rw-r----- 1 root vcap 1209 Apr  7 04:03 prometheus_ca.pem
-rw-r----- 1 root vcap 1680 Apr  7 04:03 prometheus_certificate.key
-rw-r----- 1 root vcap 1250 Apr  7 04:03 prometheus_certificate.pem

$ scp healthwatch_exporter_c* ubuntu@xxx.xxx.x.x:/home/ubuntu/workspace/prometheus/
```

### 2.1.2.  Health Watch Exporter VMs 정보 기록

- Promethues에 Exporter 정보를 등록하기 위하여 VM의 연동에 사용 될 정보를 확인 합니다.
- 이 때 필요한 정보는 pas-exporter-counter, pas-exporter-gauge, pas-exporter-timer의 IP 입니다.

```
$ bosh -d p-healthwatch2-pas-exporter-3736a1e23da111dbcb53 vms
Using environment '192.168.8.2' as client 'ops_manager'

Task 5483. Done

Deployment 'p-healthwatch2-pas-exporter-3736a1e23da111dbcb53'

Instance                                                        Process State  AZ          IPs           VM CID                                   VM Type  Active  Stemcell
bosh-deployments-exporter/6a375771-b3ea-4f18-a122-a670f553329f  running        pivotal-az  192.168.8.29  vm-a65b6f67-19fb-4a42-bb67-34e3e17a5915  small    true    bosh-vsphere-esxi-ubuntu-xenial-go_agent/621.101
bosh-health-exporter/92229e21-de78-460b-9cf3-320822e1c303       running        pivotal-az  192.168.8.28  vm-07791f20-4151-4100-8093-b089368b1771  small    true    bosh-vsphere-esxi-ubuntu-xenial-go_agent/621.101
cert-expiration-exporter/d95a2a3d-c6b5-4995-bb1f-2e0367753516   running        pivotal-az  192.168.8.30  vm-421b9885-cc12-452b-80de-89075b333a14  micro    true    bosh-vsphere-esxi-ubuntu-xenial-go_agent/621.101
pas-exporter-counter/23b7b5c9-b644-4a83-a524-a1f4df5d134e       running        pivotal-az  192.168.8.23  vm-ef1e5401-f07f-4f78-85d1-e7d1bd0ee1e0  medium   true    bosh-vsphere-esxi-ubuntu-xenial-go_agent/621.101
pas-exporter-gauge/fde2552d-ffbf-4112-8e6c-b474f199c630         running        pivotal-az  192.168.8.25  vm-615e9bbb-2e86-4d18-9ca6-feb791d1bb6d  xlarge   true    bosh-vsphere-esxi-ubuntu-xenial-go_agent/621.101
pas-exporter-timer/5e9fbdfd-1be3-44dc-8a6d-e98cb2b6b677         running        pivotal-az  192.168.8.26  vm-edc2f13a-7064-4d26-8dab-f955d1d73757  medium   true    bosh-vsphere-esxi-ubuntu-xenial-go_agent/621.101
pas-sli-exporter/b21c8d8f-5d58-4e4b-8bff-49e55870610f           running        pivotal-az  192.168.8.27  vm-fa40f3c7-c0e1-4259-b0a8-d6896e6e08a1  small    true    bosh-vsphere-esxi-ubuntu-xenial-go_agent/621.101
```

### 2.1.3.  Promethues  Config  설정 후 재 실행

- Prometheus Config  파일에 아래 설정을 추가

```
- job_name: leedh-foundation
  metrics_path: /metrics
  scheme: http
  static_configs:
    - targets:
      - "192.168.8.23:9090“ # 변경 필요
      - "192.168.8.25:9090“ # 변경 필요
      - "192.168.8.26:9090“ # 변경 필요
  tls_config:	  
    server_name: hwexporter
    ca_file: "/home/ubuntu/workspace/prometheus/healthwatch_exporter_ca.pem“ # 변경 필요
    cert_file: "/home/ubuntu/workspace/prometheus/healthwatch_exporter_certificate.pem“ # 변경 필요
    key_file: "/home/ubuntu/workspace/prometheus/healthwatch_exporter_certificate.key“ # 변경 필요
```

- Prometheus server 재 실행

```
$ sudo  systemctl reload prometheus.service
```

### 2.1.4.  Promethues에 Metrics이 정상적으로 들어오는지 확인

- http://{Promethues_URL}:9090/targets
- 아래에 등록한 Exporter IP로 UP 상태를 확인 합니다.

![exporter-1][exporter-1]

[exporter-1]:./images/exporter-image-1.PNG


### 2.1.5. Grafana UI의 Datasource에 Promtheus  등록

- Grafana UI에 Promethues를 Datastore에 입력하고 [Save & Test] 버튼을 클릭 합니다.

![exporter-2][exporter-2]

[exporter-2]:./images/exporter-image-2.PNG


### 2.1.6. Grafana UI에서 TAS Exporter에서 긁어온 Prometheus Metrics 확인

![exporter-3][exporter-3]

[exporter-3]:./images/exporter-image-3.PNG
