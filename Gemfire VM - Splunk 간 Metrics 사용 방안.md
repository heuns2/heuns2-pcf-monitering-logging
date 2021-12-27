# Gemfire VM - Splunk 간 Metrics 
- Splunk를 통해 연동 가능 한 주요 Gemfire VM의 Metrics와 임계치에 대해 설명합니다.
- 모든 Tanzu Application Service Platform의 Metrics의 구성은 동일하며 name(metrics 명칭), value(값), unit(단위)를 바탕으로 ELK, Splunk, Promethues 등에서 Query를 짤 수 있습니다.
- Splunk의 장점은 사전에 별도의 Filter를 생성 할 필요 없이 동적으로 Query를 실행하여 Dashboard를 손쉽게 만들 수 있습니다.

## 1. 주요 Gemfire VM Capacity 측정 내용

```
1. 모든 Locactor, Server Member가 떨어지지 않고 실행 중인 상태 인지 확인
2. 각 Cacher Server에 대해 Memory 사용률이 50~65% 인지 확인
3. 모든 Locactor, Server 각 각에 대해 재실행이 빈번하게 일어나는지 확인
4. Region, Server에 대한 운영이 가능한 상태인지 확인 Gets/Puts/avg latency가 ms 단위
5. JVM Palus와 Wait Thread는 나타 날 수 있지만 일시적인지 확인
6. OQL, Query를 직접적으로 사용할 경우 Query에 대한 Metrics을 확인  
```


## 2. Splunk Gemfire VM Metrics 
- Splunk Query 통해 확인이 가능한 주요 Metrics을 설명 합니다. 

```
member.MaxMemory: Memory의 사용 가능한 -Xms 최대 Heap 사용량
member.CurrentHeapSize, member.UsedMemory: 동일한 값으로 Member의 Heap 사용량
member.UsedMemoryPercentage: Member의 Heap 사용률 퍼센테이지, 50% ~ 65% 사이를 권고
member.TotalPrimaryBucketCount: Cache Server가 실제 가지고 있는 Bucket의 수
member.TotalBucketCount: Cache Server가 실제 가지고 있는 Bucket과 Redundant Data의 수, 분산 되지 않는 경우 reblance가 필요
serviceinstance.MemberCount: 현재 참여하고 있는 모든 Member의 수
member.JVMPauses: Member의 JVM 일시 정지 수, 오래 지속 될 경우 확인이 필요
member.HostCpuUsage: Member 별로의 CPU 사용률, 75%, 85%에 대한 임계치
member.GarbageCollectionCount: CMS GC Count
member.TotalFileDescriptorOpen: 전체 FileDescriptor 수
member.FileDescriptorRemaining: 사용 중인 FileDescriptor 수, TotalFileDescriptorOpen를 역전하면 장애 발생
system_healthy: Member VM의 UP/DOWN 상태를 확인
region.{REGION}.EntrySize: Region 별 Key / Value Size
region.{REGION}.EntryCount: Region 별 Key / Value의 Count
system.memory.percent: VM 전체 Memory 사용률
system.cpu.percent: VM 전체 CPU 사용률
system.persistence.disk.percent: VM 전체 persistence disk 사용률
```

## 3. gfsh Gemfire VM Metrics
- Splunk를 통해 확인이 불 가능한 주요 Metrics을 설명 합니다.

```
$ show metrics --region=/{REGION_NAME}
numBucketsWithoutRedundancy: 0 <<< Redundant 설정을 하였을때 해당 수치가 0이 아닐 경우 Rebalnce가 필요 할 수 있음
```

## 4. 오류 Metrics
- Statics File을 통해 VSD 분석 결과 Gemfire VM에서 방출 되는 Metric의 Unit 중 잘 못된 부분에 대해서 설명 합니다.

```
region.{REGION}.EntrySize: megabyte -> byte
system_disk_persistent_read_time: ms -> ns
system_disk_persistent_io_time: ms -> ns
diskstore.DiskWritesAvgLatency: ms -> ns
```

## 5. Splunk Query 결과 Sample

- 2 섹션(Splunk Gemfire VM Metrics)에서 확인 가능한 Metrics에 대해서 몇 가지 Query Sample에 대한 기록입니다, 모든 Metrics은 아래와 비슷한 유형으로 Dashboard를 만들 수 있습니다.

```
index="$index_name" deployment="$Gemfire VM Deployment Name" job="server" name="system.mem.percentage" | timechart min(value) as min by job_index

- index: 수집 대상의 splunk index 명
- deployment: $ bosh vms 명령에서의 deployment name
- job: VM Level의 Job 명
- name: 추출 대상의 Metrics Name (Splunk Gemfire VM Metrics에서 작성한 모든 Metrics 정보를 치환하면 값이 나타납니다.)
- timechart: 시간 별로 그래프 생성
- value: name에서 추출한 Metrics의 Value 값을 표시

```

- 위 Query에 대한 결과


![splunk-1][splunk-image-1]


- 위 Query를 기반하여 다른 Metrics을 통해 생성한 Dashboard (query에서 name만 변경하여 사용 할 수 있습니다.)

![splunk-2][splunk-image-2]

![splunk-3][splunk-image-3]


[splunk-image-1]:./images/splunk-image-1.png
[splunk-image-2]:./images/splunk-image-2.png   
[splunk-image-3]:./images/splunk-image-3.png











