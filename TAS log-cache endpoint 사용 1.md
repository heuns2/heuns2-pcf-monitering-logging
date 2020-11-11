
## TAS log-cache endpoint 사용 1

- PAS Platform의 내부 System endpoint를 사용하여 PAS의 주요 지표를 뽑는 방법에 대해 설명합니다.
- 우선 PAS Platform의 내부 Endpoint의 확인하는 URL은 api.sys.{DOMAIN}로 확인이 가능합니다.
	- 해당 Endpoint 중 사용 대상은  "https://log-cache.sys.{DOMAIN}" 입니다.
- 해당 방법은 PAS v2.8.x 이 후 부터 가능할 것 같습니다. 원리는 v2.8.x에서 모든 VM 내부에 Prometheus 연동에 대한   prom_scraper 기능이 활성화 되는데 이를 사용하여 Prometheus Query를 실행하여 Pivotal Platform의 Metrics를 수집하는 방법입니다.
- log-cache Endpoint는 Memory 상에 저장 된 Metrics를 추출하는 기능을 가지고 있습니다.
- 해당 방법은 p-metrics v2.0의  custom metrics 사용과 유사합니다.

### 1. Log-Cache Endpoint 사용
- 기본적으로 Endpoint를 사용하여 Metrics을 추출 할 때 cf login token이 필요 합니다.

- log-cache Endpoint 상태 확인

```
$ curl https://log-cache.sys.{DOMAIN}/api/v1/info
{"version":"2.6.1","vm_uptime":"742159"}
```

- log-cache Endpoint로 Health Watch Disk Chunk 추출

```
$  curl -G "https://log-cache.sys.{DOMAIN}/api/v1/query" --data-urlencode 'query=Diego_AvailableFreeChunksDisk{source_id="healthwatch-forwarder"}[5m]' -H "Authorization: $(cf oauth-token)" | jq

# 결과
{
  "status": "success",
  "data": {
    "resultType": "matrix",
    "result": [
      {
        "metric": {
          "ChunkCalculationValue": "6144",
          "deployment": "p-isolation-segment-4c769f595ec377c37ae5",
          "foundation": "PPMES",
          "index": "74c9caf5-590b-421a-b7f9-1e6b73de385a",
          "instance_id": "105cff5f-7a09-47ed-9dc9-1bfb5e2d740a",
          "ip": "",
          "job": "healthwatch-forwarder",
          "origin": "healthwatch",
          "source_id": "healthwatch-forwarder"
        },
        "values": [
          [
            1584065220,
            "33"
          ],
          [
            1584065340,
            "33"
          ]
        ]
      },
      {
        "metric": {
          "ChunkCalculationValue": "6144",
          "deployment": "cf-41664204d7baf8f010da",
          "foundation": "PPMES",
          "index": "74c9caf5-590b-421a-b7f9-1e6b73de385a",
          "instance_id": "105cff5f-7a09-47ed-9dc9-1bfb5e2d740a",
          "ip": "",
          "job": "healthwatch-forwarder",
          "origin": "healthwatch",
          "source_id": "healthwatch-forwarder"
        },
        "values": [
          [
            1584065220,
            "255"
          ],
          [
            1584065340,
            "255"
          ]
        ]
      }
    ]
  }
}
```

### 2. 기타 Pivotal Platform의 Metrics 추출
- Platform의 주요 컴포넌트의 설정 파일 부분에는 아래와 같은 Metrics를 추출하여 계산하는 부분이 존재하고 있습니다. 예시는 Diego VM의 상태 체크를 맡는 rep입니다.
- /var/vcap/jobs/rep/config/indicators.yml
- 아래 파일에는 promql: min_over_time(CapacityRemainingMemory{source_id="rep"}[5m]) / 1024이라는 예시가 존재합니다. 해당 promql을 통해 log-cache endpoint로 값을 추출 할 수 있습니다.

```
spec:
  product:
    name: diego
    version: latest

  indicators:
  - name: capacity_remaining_memory
    promql: min_over_time(CapacityRemainingMemory{source_id="rep"}[5m]) / 1024
    documentation:
      title: Diego Cell - Remaining Memory Available - Overall Remaining Memory Available
      description: |
        Remaining amount of memory in MiB available for this Diego cell to allocate to containers.

        Use: Can indicate low memory capacity overall in the platform. Low memory can prevent app scaling and new deployments. The overall sum of capacity can indicate that you need to scale the platform. Observing capacity consumption trends over time helps with capacity planning.

        Origin: Firehose
        Type: Gauge (Integer in MiB)
        Frequency: 60 s
      recommended_response: |
        1. Assign more resources to the cells
        2. Assign more cells.
```

- log-cache endpoint 사용 예시

```
$ curl -G "https://log-cache.sys.{DOMAIN}/api/v1/query" --data-urlencode 'query=min_over_time(CapacityRemainingMemory{source_id="rep"}[5m]) / 1024' -H "Authorization: $(cf oauth-token)" | jq

# 결과
{
  "status": "success",
  "data": {
    "resultType": "vector",
    "result": [
      {
        "metric": {
          "deployment": "cf-41664204d7baf8f010da",
          "index": "52fd989b-bff8-4c2e-ba62-db5a492a4841",
          "instance_id": "52fd989b-bff8-4c2e-ba62-db5a492a4841",
          "ip": "172.18.92.121",
          "job": "diego_cell",
          "origin": "rep",
          "product": "Pivotal Application Service",
          "source_id": "rep",
          "system_domain": "sys.{DOMAIN}",
          "zone": "az2"
        },
        "value": [
          1584065811,
          "7.6640625"
        ]
      },
      {
        "metric": {
          "deployment": "cf-41664204d7baf8f010da",
          "index": "7492ad9b-350e-411a-8136-5e240482ae8d",
          "instance_id": "7492ad9b-350e-411a-8136-5e240482ae8d",
          "ip": "172.18.92.139",
          "job": "diego_cell",
          "origin": "rep",
          "product": "Pivotal Application Service",
          "source_id": "rep",
          "system_domain": "sys.{DOMAIN}",
          "zone": "az3"
        },
        "value": [
          1584065811,
          "7.9453125"
        ]
      },...생략
}
```

- 추출 대상의 Metrics은https://docs.cloudfoundry.org/running/all_metrics.html입니다.
