# PAS Log and Metric Agent Architecture VM 내부 Promcraper

- Loggregator, Doppler VM 비 공유 Architecture 중 **Metrics**에 대한 사용 구성 방법을 설명합니다.
- 기존 Loggregator에서 방출 되는 모든 PAS System Metrics는 Loggregator Firehose 또는 Nozzle을 통해 출력되었습니다. 하지만 변경 된 신규 Architecture(PAS 2.8.x)에서는 Pivotal Platform의 모든 VM에 배치 되는 **metrics-agent** Job의 설정을 통하여 Loggregator Firehose를 거치지 않고 외부에서 Metrics를 TLS 통신으로 직접 접근 가능하도록 설계 되었습니다.
- Metrics를 직접 접근하는데 사용된 기술은 Prom_Scraper이며 Prometheus Exposition 기반으로 HOST_IP/metrics Endpoint 활성화되어 Metrics를 추출 할 수 있습니다.
- **metrics-agent**는 기존 Loggregator Architecture에서의 **loggregator-agent**와 비슷한 기능을 하며 PAS 2.9.x 부터는 firehose v1, v2를 disable 시켜 loggregator-agent을 비활성화 할 수 있습니다.
- 이전 Dojo에서의 Opensource Prometheus 배포와 다르게 해당 기능을 사용하여 Prometheus_Firehose_Exporter를 전체 VM에 배포 할 필요 없이 Endpoint만을 사용하여 이전과 같은 전체 Metrics를 추출 가능 합니다.


## 1. VM에서의 Prom_Scraper 구성 확인

### 1.1. Jobs
- Pivotal Platform의 모든 VM의 Metrics 방출 대상(metrics-agent, loggregator-agent, metrics-discovery-registrar) Job 아래에는 Prom_Scraper.config 파일이 존재합니다.

```
/var/vcap/data/jobs/prom_scraper/config/prom_scraper_config.yml
/var/vcap/data/jobs/metrics-discovery-registrar/config/prom_scraper_config.yml
/var/vcap/data/jobs/loggr-syslog-agent/config/prom_scraper_config.yml
/var/vcap/data/jobs/metrics-agent/config/prom_scraper_config.yml
/var/vcap/data/jobs/loggr-forwarder-agent/config/prom_scraper_config.yml
/var/vcap/data/jobs/loggregator_agent/config/prom_scraper_config.yml
```

- prom_scraper_config.yml 예시

```
$ cat prom_scraper_config.yml
---
port: 14827
source_id: "metrics-agent"
instance_id: 0e4cc47a-abc1-4360-b3f7-46ba7fc18af1
scheme: https
server_name: metrics_agent_metrics
```

- metrics-agent config 파일 예시

```
---
processes:
- name: metrics-agent
  executable: "/var/vcap/packages/metrics-agent/metrics-agent"
  unsafe:
    unrestricted_volumes:
    - path: "/var/vcap/jobs/*/config/prom_scraper_config.yml"
      mount_only: true
  ephemeral_disk: true
  env:
    AGENT_PORT: '3461'
    AGENT_CA_FILE_PATH: "/var/vcap/jobs/metrics-agent/config/certs/grpc_ca.crt"
    AGENT_CERT_FILE_PATH: "/var/vcap/jobs/metrics-agent/config/certs/grpc.crt"
    AGENT_KEY_FILE_PATH: "/var/vcap/jobs/metrics-agent/config/certs/grpc.key"
    AGENT_TAGS: deployment:cf-176c7ba1fff553e430bb,instance_group:diego_cell,index:0e4cc47a-abc1-4360-b3f7-46ba7fc18af1
    CONFIG_GLOBS: "/var/vcap/jobs/*/config/prom_scraper_config.yml"
    METRICS_EXPORTER_PORT: '14726'
    METRICS_PORT: '14827'
    METRICS_CA_FILE_PATH: "/var/vcap/jobs/metrics-agent/config/certs/metrics_ca.crt"
    METRICS_CERT_FILE_PATH: "/var/vcap/jobs/metrics-agent/config/certs/metrics.crt"
    METRICS_KEY_FILE_PATH: "/var/vcap/jobs/metrics-agent/config/certs/metrics.key"
    WHITELISTED_TIMER_TAGS: source_id,deployment,job,index,ip
    METRICS_TARGETS_FILE: "/var/vcap/data/metrics-agent/metric_targets.yml"
    ADDR: 172.28.106.203
    INSTANCE_ID: 0e4cc47a-abc1-4360-b3f7-46ba7fc18af1
    SCRAPE_CA_CERT_PATH: "/var/vcap/jobs/metrics-agent/config/certs/scrape_ca.crt"
    SCRAPE_CERT_PATH: "/var/vcap/jobs/metrics-agent/config/certs/scrape.crt"
    SCRAPE_KEY_PATH: "/var/vcap/jobs/metrics-agent/config/certs/scrape.key"
```

- **Metrics-Agent의 14827  Port를 통해 Metrics를 Prometheus 형태의 Metrics으로 변환하고 METRICS_EXPORTER_PORT: '14726'를 통해 외부 Local, Container, VM 등에서 해당 Metrics에 접근이 가능 하게 됩니다.**


## 2. Prometheus Exposition 사용

- 우선 Promethues 기반의 Metrics을 처리하는 Tool에서는 모두 사용 가능하다고 추측하지만 Pivotal Docs에서는 Grafana, Infulx Telegraf의 사용 방법에 대해 설명 되어 있으며... 추후 Dynatrace가 제거 될 예정이라면 고려 할 만한 방법인 것 같습니다.
- 외부에서 위에서 언급한 METRICS_EXPORTER_PORT: '14726'에 접근하는 방법은 아래와 같습니다.
- 본 문서에서는 Telegraf의 사용 방법을 예시로 합니다. Telegraf은 Logstash 처럼 서로 다른 플랫폼 Formation간 데이터를 수집하고, 가공하여 전달해주는 기능을 하는 것 같습니다.
- Telegraf를 사용하게 되면 PAS의 NATS를 컴포넌트를 통해 모든 VM에 대한 정보를 가져 올 수 있다고 합니다.

### 2.1. telegraf.config

- Input은 telegraf이 수집할 데이터에 대한 Input 방식 값을 설정 하는 것 입니다. 이때 필요 한 값은 각 **VM_HOST_IP:METRICS_EXPORTER_PORT/metrics**  Prometheus 데이터를 수집해야하며 TLS 통신에 사용 될 scrape certificate 정보를 입력해야 합니다.

```
[inputs.prometheus]
tls_ca="/home/vcap/app/certs/scrape_ca.crt"
tls_cert="/home/vcap/app/certs/scrape.crt"
tls_key="/home/vcap/app/certs/scrape.key"
insecure_skip_verify=true
urls=[ "https://172.28.106.203:14726/metrics" ]
```

- Output은 수집 된 Prometheus 형태의 Metrics을 어느 플랫폼으로 보낼 것인지에 대한 설정 값입니다. 예시에서는 ELK로 보냈지만 output 방식에 Splunk 등 많은 플랫폼을 지원하는 것 같습니다.

```
[[outputs.elasticsearch]]
  urls = [ "http://172.18.80.226:9200" ] # required.
  timeout = "60s"
  enable_sniffer = false
  health_check_interval = "10s"
  index_name = "my-computer-metric-%Y.%m.%d" # required.
  manage_template = false
  template_name = "telegraf"
  overwrite_template = false
``` 

- 수집 대상의 Interval, flush 등 설정

```
[agent]
  interval = "30s"
  round_interval = true
  metric_batch_size = 1000
  metric_buffer_limit = 100000
  collection_jitter = "1s"
  flush_interval = "10s"
  flush_jitter = "1s"
  precision = ""
  hostname = ""
  omit_hostname = false
```

- 해당 설정을 바탕으로 telegraf 바이너리를 실행하면 ELK Kibana를 통해 https://172.28.106.203:14726/metrics에서 수집되는 Metrics를 확인 할 수 있습니다.

**위와 같은 설정과 완전 비슷하게 Grafana를 구성 할 수도 있습니다.**
