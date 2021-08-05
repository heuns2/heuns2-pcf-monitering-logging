# Health Watch v2.x Grafana Plugin 사용 (Main Dashboard)

- 본 문서는 Health Watch v2.x에서 특정 Plugin(Status Panel)을 통해 다수의 Metrics Query를 하나의 Panel에서 Critical, Warning, Normal을 Groups 화하여 표기하여 중앙의 한 Dashboard에서 Alert를 확인하고, 해당 Panel을 통해 문제의 Dashboard로 이동, 관리하는 방안에 대해 설명 합니다.
- 본 문서는 전체 Metrics을 확인 한 것이 아닌 부분적인 Metrics만을 확인 했으며 Value 값 만을 취급하여 표기 됩니다.
- Test 용으로 Metrics에 대한 추가적인 단위(ms, gb), 함수(avg, max, min, last)에 대한 설정이 필요 할 가능성이 있습니다.
- 본 가이드는 Health Watch v2.1.1에서 진행되었습니다. 추후 버전이 업그레이드 되면 지원이 불가능 할 수 있습니다.
- 아래 이미지 형태로 Metrics이 정상일때에는 OK 출력(초록), 일정 수준의 임계치를 넘어가면 Warning(주황), Critical(빨강)으로 표시 되며 수치가 넘어간 Metrics에 대해서만 표기 됩니다.

![grafana-1][grafana-1]

[grafana-1]:./images/grafana-1.PNG

## 1. Status Panel Plugin 설치

- 외부 통신이 되지 않는 것을 감안하여 VM의 아래 디렉토리에 Status Panel Plugin 파일을 전송하여 적용합니다.

```
# Health Watch Grafana VM에 아래 링크에서 다운로드 받은 File을 전송
https://grafana.com/api/plugins/vonage-status-panel/versions/1.0.11/download

# Health Watch Grafana VM에 SSH하여 접속하여 아래 디렉토리에 Zip 파일 압축 해제합니다.
$ cd  /var/vcap/store/grafana/grafana_plugins
$ ls  –al
total 296
drwxr-xr-x 5 root root 4096 Aug  5 05:57 .
drwx------ 4 vcap  vcap 4096 Jun  4 07:53 ..
drwxr-xr-x 3 root root 4096 Aug  5 04:29 flant-statusmap-panel
drwxr-xr-x 4 root root 4096 Jun  3 19:25 vonage-status-panel
-rw-r--r-- 1 root root 279042 Jun  3 18:05 vonage-status-panel-1.0.11.zip

# Grafana Process 재기동
$ sudo  monit restart grafana

# Grafana Process Running 상태 확인
$ sudo monit summary
The Monit daemon 5.2.5 uptime: 2h 32m

Process 'grafana'                   running
Process 'reverse-proxy'             running
Process 'userinfo-proxy'            running
Process 'bosh-dns'                  running
Process 'bosh-dns-resolvconf'       running
Process 'bosh-dns-healthcheck'      running
Process 'system-metrics-agent'      running
System 'system_localhost'           running

```

- Plugin이 올바르게 적용 되었는지 확인 합니다.

![grafana-2][grafana-2]

[grafana-2]:./images/grafana-2.PNG


## 2. Status Panel 설정

### 2.1. Status Panel 생성

- 신규 Dashboard를 생성하여 Status Panel을 생성 합니다.

![grafana-3][grafana-3]

[grafana-3]:./images/grafana-3.PNG


- Grouping 할 Panel명칭을 명시 하고 Visualization 창에서 Status Panel을 선택합니다.

![grafana-4][grafana-4]

[grafana-4]:./images/grafana-4.PNG

### 2.2. Metrics 연동

- 가져올 Metrics(Health Watch v1.x에서 제공한)과 Legend를 작성하고 + Query 버튼을 클릭하여 관리 할 대상의 모든 Metrics을 명시 합니다.
- 본 가이드에서는 LRPsRunning, ActiveLocks Metrics을 사용하여 Test를 진행 했습니다.

![grafana-5][grafana-5]

[grafana-5]:./images/grafana-5.PNG


### 2.3. Panel 설정 변경

-UI 오른쪽 Panel 설정 부분의 Options 아래 부분의 설정을 편집합니다.

-[Display 생성]

![grafana-6][grafana-6]

[grafana-6]:./images/grafana-6.PNG

1.  Alias는 Alert 발생 시 표기 되는 명칭이며 이전 5P에서 설정한 Legend 명칭과 같아야합니다.
2. Handler Type은 Number, String, Date 등 임계치에 대한 Type을 지정 할 수 있습니다.
3. Waring, Critical에 표기된 값이 넘으면 Alert가 발생 합니다.
4. Unit은 Alert 발생 시 어떠한 값으로 표기를 시킬 것인지 설정 가능 합니다.
5. Display Alias, Display Value는 모두 Alert 발생 시에만 값을 불러오도록 Waring/Critical로 사용합니다.



### 2.4. Link 설정

- Grouping 대상의 Metrics이 정확하게 어떠한 문제인지, 어떤 컴포넌트에서 에러가 발생했는지 확인 하기 위해 바로 이동을 할 수 있는 Link를 설정 합니다.
- [Link 생성]

![grafana-7][grafana-7]

[grafana-7]:./images/grafana-7.PNG


### 2.5. Panel과 Dashbord  저장 및 확인

- Panel과 Dashboard를 저장하고 Link가 올바르게 맺어졌는지 확인합니다

![grafana-8][grafana-8]

[grafana-8]:./images/grafana-8.PNG

## 3. 최종 확인 및 활용 방안

- Grouping 된 Metrics 임계치에 대한 Alert를 확인하고 장애가 있는 Alert 있는 상세 Dashboard로 이동하여 상세 내용 확인이 가능 할 것 같습니다.

![grafana-9][grafana-9]

[grafana-9]:./images/grafana-9.PNG

