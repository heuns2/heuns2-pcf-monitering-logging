# Bosh Self Metrics - Promethues 연동

- 본 문서는 Bosh Metrics과 Health Watch v2.x 또는 Prometheus에 연동하는 방법에 대해서 설명 합니다.
- Bosh를 통하여 관리되는 Platform VM들은 Runtime Config 방식을 통하여 Agent를 심거나 모든 VM에 서 실행 중인 내부 Scraper 시스템과 자동 연동을 할 수 있지만, Management 영역인 Ops Man과 Bosh는 수동으로 연동을 시켜야 합니다.
- 사전 조건으로는 Bosh(Ops Manager) Version v2.10.9 이상이여야 합니다.

## 1. Bosh에서 설정 가져오기

### 1.1. Bosh VM ssh 접속
- Ops Manager UI에 접속하여 [Bosh Director Tile] -> [Credentials] -> [Bbr Ssh Credentials]에서 private key를 복사하여 아래 Command를 통하여 Bosh에 접속 할 수 있는 Key 형태를 생성합니다.
 
```
# 예시
$ printf -- "-----BEGIN RSA PRIVATE KEY-----\nMIIEpAIBAAKCAQEAohGHVwEVNAm/\nt9AcDTjrMyKvyejWuxSYEwiLDC2O3VL5UrZLPoHtdLK2qJYExXiNBw==\n-----END RSA PRIVATE KEY-----\n" > bosh.pem

# key 파일 권한 변경
$ chmod 600 bosh.pem

# Bosh VM 접속
$ ssh -i bosh.pem bbr@{BOSH_IP}
```

### 1.2. Promethues Metrics 정보 확인
- Bosh VM에 접속 상태에서 Promethues Metrics Query 결과가 정상적으로 출력 되는지 확인합니다.

```
$ curl -k https://localhost:53035/metrics --cacert /var/vcap/jobs/loggr-system-metrics-agent/config/certs/system_metrics_agent_ca.crt --key /var/vcap/jobs/loggr-system-metrics-agent/config/certs/system_metrics_agent.key --cert /var/vcap/jobs/loggr-system-metrics-agent/config/certs/system_metrics_agent.crt

# HELP system_cpu_core_idle vm metric
# TYPE system_cpu_core_idle gauge
system_cpu_core_idle{cpu_name="cpu0",deployment="p-bosh",index="516dacbc-c326-4b23-4e32-5851ecfa2e99",ip="",job="loggr-system-metrics-agent",origin="system_metrics_agent",source_id="system_metrics_agent",unit="Percent"} 90.4071710448007
system_cpu_core_idle{cpu_name="cpu1",deployment="p-bosh",index="516dacbc-c326-4b23-4e32-5851ecfa2e99",ip="",job="loggr-system-metrics-agent",origin="system_metrics_agent",source_id="system_metrics_agent",unit="Percent"} 90.57620630603405
# HELP system_cpu_core_sys vm metric
# TYPE system_cpu_core_sys gauge
system_cpu_core_sys{cpu_name="cpu0",deployment="p-bosh",index="516dacbc-c326-4b23-4e32-5851ecfa2e99",ip="",job="loggr-system-metrics-agent",origin="system_metrics_agent",source_id="system_metrics_agent",unit="Percent"} 0.7372936447622622
system_cpu_core_sys{cpu_name="cpu1",deployment="p-bosh",index="516dacbc-c326-4b23-4e32-5851ecfa2e99",ip="",job="loggr-system-metrics-agent",origin="system_metrics_agent",source_id="system_metrics_agent",unit="Percent"} 0.7232341547844722
# HELP system_cpu_core_user vm metric
# TYPE system_cpu_core_user gauge
....생략
```

### 1.3. Key 정보 복사
- Bosh VM 내부의 아래 디렉토리에서 3종류의 CA, Cert, Key 파일을 메모장에 복사 합니다.

```
$ cd /var/vcap/jobs/loggr-system-metrics-agent/config/certs/
$ ls -al
total 20
drwxr-x--- 2 root vcap 4096 May 10 05:55 ./
drwxr-x--- 3 root vcap 4096 May 10 05:55 ../
-rw-r----- 1 root vcap 1209 May 10 05:55 system_metrics_agent_ca.crt
-rw-r----- 1 root vcap 1258 May 10 05:55 system_metrics_agent.crt
-rw-r----- 1 root vcap 1680 May 10 05:55 system_metrics_agent.key

``` 

## 2. Health Watch 설정 & 변경 사항 적용
- 위에서 가져온 Key 정보를 바탕으로 Health Watch Tile 설정을 편집합니다.
- Ops Manager UI에 접속하여 [Health Watch Tile] -> [Prometheus Configuration] -> [Additional Scrape Config Jobs Add]에 [add 버튼] 클릭 합니다.

- TSDB Scrape job에 설정 정보 입력합니다. BOSH_IP 정보는 변경이 필요 합니다.

```
job_name: director
metrics_path: /metrics
scheme: https
static_configs:
  - targets:
    - "{BOSH_IP}:53035"  
```

- TLS Config Certificate Authority 정보에 Bosh VM 내부에서가져온 [system_metrics_agent_ca.crt] 값을 입력합니다.
- TLS Config Certificate and Private Key 정보에 Bosh VM 내부에서가져온 [system_metrics_agent.crt, system_metrics_agent.key] 정보를 입력 합니다.
- TLS Config Server Name에 [system-metrics]를 입력 합니다.
- TLS Config Skip SSL Validation를 Check 하고 [SAVE] 버튼을 클릭합니다.
- Health Watch v2.x Tile을 Apply Change 합니다.

## 3. Grafana Dashborad 확인

- Grafana UI에 접근하여 Explore에 아래 Sample 쿼리를 실행 시켜 정상적으로 Bosh의 Metrics이 수집 되는지 확인 합니다.

```
system_cpu_user{origin="system_metrics_agent", deployment=~"p-bosh"} 
```



