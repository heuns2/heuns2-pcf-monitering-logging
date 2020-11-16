# Gemfire - VSD Tools 사용
- Gemfire는 Apache Geode를 기반으로 만든 Pivotal의 Cache 저장소 시스템입니다.
- Pivotal Gemfire VMs를 설치하게 되면 내부적으로 Pulse가 설치되고 Cache를 저장하는 Region 생성 중 옵션을 통해 Metrics을 방출 할 수 있으며, 다양한 모니터링 Tool을 통해 Metrics을 확인 할 수 있지만 이것은 부분적이며 더 상세하게 확인을 하기 위해서는 VSD Tool을 사용해야 합니다.
- Partition, Replica Region 별로의 GET, PUT, REMOVE 에 대한 상세 비율, Java로 실행 되는 Gemfire Server Process의 JVM 뿐만 아닌 Native 영역의 Memory 사용률 뿐만 아닌 Gemfire Client와의 Message Queued, Processing Time을 보여줍니다.
- 운영 중 주로 확인 하였던 지표
	- 75%에서 GC가 발생하지만 주로 Full GC를 통해 Memory 해제가 되지 않아 Scale UP을 고려
	- Region 별 GET, PUT, POST, DELETE 에 대한 비중을 확인하여 Partition, Replica Type 정의를 고려, Region Type에 따라서 read, write 시간에 대한 차이가 있습니다.
	- Region 별 Eviction Destory Data의 비중을 확인
	- Partition Region의 경우 Redendent 의 수를 확인

## 1. 사용 방안

### 1.1. Region 속성 정의
- gfsh 또는 API를 통하여 Region을 생성 할 경우 아래의 속성 추가가 필요합니다. --enable-statistics=true 속성을 통해 약간의 Memory는 추가되지만 Region에 대한 상세 Metrics이 추출 됩니다.

```
$ create region --name=test --enable-statistics=true
```

### 1.2. .GFS File
- VSD에 File을 Upload 하기 위해서는 .gfs 파일이 필요합니다. 해당 파일은 Cache Server VM의 /var/vcap/log/gemfire-server/server/ 디렉토리에 statistics-*.gfs위 명칭으로 위치 하고 있습니다. 해당 파일을 VSD가 설치 되어 있는 곳으로 위치 시킵니다.
- VSD를 실행하고 $ vsd라고 명령을 치고 해당 File을 Upload 합니다.


### 1.3. DATA 확인 Sample Image

- Region의 GET Ratio (GET이 많을 경우 Replica Type의 Region으로 생성을 권고)

![gfs-1][gfs-image-1]

- Region의 PUT Ratio (PUT이 많을 경우 Partition Type Region으로 생성을 권고)

![gfs-2][gfs-image-2]

- Cache Server VM의 CMS GC 후 Memory 변화 (GC  이후 Memory 변화율)

![gfs-3][gfs-image-3]

- Region의 LRU Destory 비율 (만료 설정으로 인하여 Destory 되는 Cache DATA를 확인 - Default Heap 75%이상 되면 가장 마지막 Data가 삭제)

![gfs-3][gfs-image-4]


[gfs-1]:./images/gfs-image-1.png
[gfs-2]:./images/gfs-image-2.png
[gfs-3]:./images/gfs-image-3.png
[gfs-4]:./images/gfs-image-4.png

