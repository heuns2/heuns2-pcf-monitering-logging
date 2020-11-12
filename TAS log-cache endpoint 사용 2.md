## Pivotal Platform Prometheus Endpoint 사용 (고급)
- 이전 Tracker를 기반으로 정리한 내용입니다. 추후 Prometheus - Grafana 연동 시 유용하게 사용 가능해 보입니다. 
- Tile 또는 Open Source를 통해 Default로 설치 시 자동으로 UI는 그려지겠지만, 여러 Foundation에서의  Pivotal Platform의 모든 VM에 Firehose Release가 설치 됨으로 모든 VM에서의 부하량이 증가 할 수 있습니다.
- Log Cache Endpoint을 통해 Metrics 접근은 Endpoint 호출 > source id 접근 -> metrics 접근 순 입니다. metrics 접근 간의 mysql로 따지면 where 절을 추가하여 특정 metrics를 추출 합니다.

### 1. Log Cache Endpoint를 통해 Customize 가능한 Data의 source_id 값 확인
- 아래 명령어를 통해 PAS에서 사용 가능한 전체 Metrics 정보의 상위 Keyword인 source_id를 추출가능 합니다.

```
$ curl -G "https://log-cache.sys.{DOMAIN}/api/v1/meta"  -H "Authorization: $(cf oauth-token)" | jq

# 결과 값 예시 일 부분
    "rep": {
      "count": "100000",
      "expired": "930435",
      "oldestTimestamp": "1584334406652976684",
      "newestTimestamp": "1584335545553816523"
    },
    "reverse_log_proxy": {
      "count": "8385",
      "oldestTimestamp": "1584323559227659649",
      "newestTimestamp": "1584335543494671359"
    },
    "reverse_log_proxy_gateway": {
      "count": "100000",
      "expired": "125176",
      "oldestTimestamp": "1584330225761015123",
      "newestTimestamp": "1584335543194141473"
    },
    "route_emitter": {
      "count": "100000",
      "expired": "638767",
      "oldestTimestamp": "1584333866607469951",
      "newestTimestamp": "1584335545573443065"
    },
    "routing_api": {
      "count": "39757",
      "oldestTimestamp": "1584323172166844993",
      "newestTimestamp": "1584335545412258223"
    },
    "server": {
      "count": "100000",
      "expired": "162769",
      "oldestTimestamp": "1584330733300522363",
      "newestTimestamp": "1584335544550935342"
    },
    "silk-daemon": {
      "count": "69481",
      "oldestTimestamp": "1584322990460623734",
      "newestTimestamp": "1584335544487026268"
    },
    "ssh_proxy": {
      "count": "14025",
      "oldestTimestamp": "1584323848459303323",
      "newestTimestamp": "1584335538459497972"
    },
    "syslog_agent": {
      "count": "100000",
      "expired": "5237963",
      "oldestTimestamp": "1584335325801370812",
      "newestTimestamp": "1584335545635054612"
    },
    "system_metric_scraper": {
      "count": "100000",
      "expired": "7558",
      "oldestTimestamp": "1584324678179507401",
      "newestTimestamp": "1584335538450570762"
    },
    "system_metrics_agent": {
      "count": "100000",
      "expired": "5855197",
      "oldestTimestamp": "1584335341230919978",
      "newestTimestamp": "1584335536206703413"
    },
    "traffic_controller": {
      "count": "10784",
      "oldestTimestamp": "1584323574579485953",
      "newestTimestamp": "1584335541725178450"
    },
    "uaa": {
      "count": "100000",
      "expired": "330728",
      "oldestTimestamp": "1584332617023038971",
      "newestTimestamp": "1584335545373539529"
    },
    "udp_forwarder": {
      "count": "100000",
      "expired": "345166",
      "oldestTimestamp": "1584332707068495161",
      "newestTimestamp": "1584335545340468967"
    },
    "vxlan-policy-agent": {
      "count": "100000",
      "expired": "23547",
      "oldestTimestamp": "1584325303359790849",
      "newestTimestamp": "1584335544626974484"
    }
```

- 값 설명
	- 결과 값 중 상위인 vxlan-policy-agent가 추 후 metrics를 검색 할 때 사용 되는 source_id 값 입니다.
	- count는 timestamp 기간 내의 metrics의 수 입니다.
	- log cache endpoint의 대상 metrics은 모든 데이터를 DB 저장하지 않고 Memory에 저장 후 사용 됩니다. 추 후 기타 서비스를 통해 저장해야 합니다.


### 2. source_id 값을 통해  Metrics 정보 추출
- source_id를 통하여 추출한 Metrics의 전체 정보를 출력 합니다.

```
$ curl -G "https://log-cache.sys.{DOMAIN}/api/v1/read/vxlan-policy-agent" -H "Authorization: $(cf oauth-token)"  | jq

# 결과 값 일부 발췌
      {
        "timestamp": "1584325591519124959",
        "source_id": "vxlan-policy-agent",
        "instance_id": "",
        "deprecated_tags": {},
        "tags": {
          "__v1_type": "ValueMetric",
          "deployment": "p-isolation-segment-4c769f595ec377c37ae5",
          "index": "368120c6-8348-4c91-a9d5-f781d98d15df",
          "ip": "172.18.93.155",
          "job": "isolated_diego_cell",
          "origin": "vxlan-policy-agent",
          "placement_tag": "pkmes-is",
          "product": "Pivotal Isolation Segment",
          "system_domain": "sys.{DOMAIN}"
        },
        "gauge": {
          "metrics": {
            "policyServerPollTime": {
              "unit": "ms",
              "value": 2.052509
            }
          }
        }
```
- 결과 값 설명
	- gauge 영역: 실제 가져올 metrics의 값입니다.
	- tag 영역: metrics를 가져 오기 위한 query 값 중 where 문에 해당 됩니다. 

- 만약 위 값에서 policyServerPollTime의 value를 가져오기 위해서는 아래의 query가 필요 합니다.

```
# curl -G "https://log-cache.sys.{DOMAIN}/api/v1/query" --data-urlencode 'query=policyServerPollTime{source_id="vxlan-policy-agent",job="isolated_diego_cell"}' -H "Authorization: $(cf oauth-token)" | jq

# 결과 값
      {
        "metric": {
          "__v1_type": "ValueMetric",
          "deployment": "p-isolation-segment-4c769f595ec377c37ae5",
          "index": "d27eddc8-9d08-42c6-9e27-9120c3d81f8e",
          "ip": "172.18.93.156",
          "job": "isolated_diego_cell",
          "origin": "vxlan-policy-agent",
          "placement_tag": "pkmes-is",
          "product": "Pivotal Isolation Segment",
          "source_id": "vxlan-policy-agent",
          "system_domain": "sys.{DOMAIN}"
        },
        "value": [
          1584336033,
          "1.526036"
        ]
      }
```

## meta의 결과 값 중 주요 property 설명
- Platform 상에서 수집 되고 있는 전체 Metrics 입니다.

```
Source                         Source Type  Count   Expired  Cache Duration
auctioneer                     platform     24811   0        3h38m40s
bbs                            platform     100000  19444    2h32m50s
bosh-dns-adapter               platform     66258   0        3h25m57s
bosh-system-metrics-forwarder  platform     66215   0        3h32m25s
cc                             platform     100000  39278    2h28m31s
doppler                        platform     100000  76435    1h55m43s
file_server                    platform     14823   0        3h26m0s
forwarder_agent                platform     100000  4666998  4m33s
garden-linux                   platform     65246   0        3h38m38s
gorouter                       platform     100000  4778945  4m26s
grootfs                        platform     6849    0        3h29m44s
healthwatch-forwarder          platform     20209   0        3h31m39s
iptables-logger                platform     65119   0        3h38m37s
leadership-election            platform     100000  9445     3h13m45s
locator                        platform     100000  23092    2h53m33s
locket                         platform     38178   0        3h32m25s
log-cache                      platform     100000  580803   30m35s
log-cache-cf-auth-proxy        platform     100000  350593   46m16s
log-cache-gateway              platform     100000  341778   49m34s
log-cache-nozzle               platform     100000  367234   45m15s
loggr-syslog-binding-cache     platform     100000  19348    3h5m10s
metric_registrar               platform     26588   0        3h38m32s
metrics_discovery_registrar    platform     100000  4825346  4m25s
metron                         platform     100000  5570612  3m37s
netmon                         platform     72949   0        3h24m22s
p-cloudcache                   platform     202     0        3h23m28s
p-mysql                        platform     80067   0        3h35m22s
prom_scraper                   platform     100000  5477382  3m50s
rep                            platform     100000  979606   18m58s
reverse_log_proxy              platform     8777    0        3h29m0s
reverse_log_proxy_gateway      platform     100000  135757   1h28m37s
route_emitter                  platform     100000  671998   27m30s
routing_api                    platform     41578   0        3h35m34s
server                         platform     100000  174508   1h20m2s
silk-daemon                    platform     72616   0        3h38m33s
ssh_proxy                      platform     14705   0        3h24m18s
syslog_agent                   platform     100000  5494133  3m39s
system_metric_scraper          platform     100000  12711    3h1m10s
system_metrics_agent           platform     100000  6129653  3m26s
traffic_controller             platform     11288   0        3h28m52s
uaa                            platform     100000  349945   48m44s
udp_forwarder                  platform     100000  364906   47m21s
vxlan-policy-agent             platform     100000  29084    2h5
```

- bosh-system-metrics-forwarder: Bosh가 수집하는 System Metrics으로써 Bosh를 통해 설치 한 모든 VM들의 System Metric(CPU/Memory/Disk 사용률)를 가져 올 수 있습니다.
- healthwatch-forwarder: Health Watch에 표시 되는 모든 지표(Chunk, cf cli의 오류 등)를 사용 할 수 있습니다.
- cc: Cloud Controller 관련 Metrics를 수집 할 수 있습니다.
- gorouter 관련 모든 Metrics를 수집 할 수 있습니다.
- rep: Diego Container 관련 Metrics를 수집 할 수 있습니다.
