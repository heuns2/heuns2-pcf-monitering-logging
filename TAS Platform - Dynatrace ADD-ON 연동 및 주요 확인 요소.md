# TAS Platform - Dynatrace ADD-ON 연동 및 주요 확인 요소
- 사전 준비 사항
	- Dynatrace One Agent Release: https://github.com/Dynatrace/bosh-oneagent-release/releases
	- Dynatrace Runtime Config

## 1. TAS - Dynatrace OneAgent 연동
- Runtime Config를 바탕으로 Dynatrace를 연동하게 되면 모든 VM에 Agent가 설치되며 자동으로 fullstack, infra mode에 기반하여 VM에 대한 일반적인 Metrics 부터 Process의 상세 정보, App에 대한 JVM, Request & Response 부터 Code Level의 Metrics까지 수집이 됩니다.

### 1.1. 설치 방법

#### 1.1.1. Upload Release
- Download한 Release를 Bosh Director에 Upload 합니다.

```
$ bosh upload-release dynatrace-oneagent-1.3.3.tgz
Using environment '10.0.16.5' as client 'ops_manager'

######################################################## 100.00% 304.01 KiB/s 0s
Task 3625

Task 3625 | 05:36:02 | Extracting release: Extracting release (00:00:00)
Task 3625 | 05:36:02 | Verifying manifest: Verifying manifest (00:00:00)
Task 3625 | 05:36:03 | Resolving package dependencies: Resolving package dependencies (00:00:00)
Task 3625 | 05:36:03 | Creating new jobs: dynatrace-oneagent/dddd5bb8a9ba332aa38905400fd22f6503c55ac40e92f3c2868a50f764cc4c96 (00:00:00)
Task 3625 | 05:36:03 | Creating new jobs: dynatrace-oneagent-windows/82e326937cef2fd27501e804cd1d246a1ad8e4c61f9c25c95bfc6c2e1f4980a9 (00:00:00)
Task 3625 | 05:36:03 | Release has been created: dynatrace-oneagent/1.3.3 (00:00:00)

Task 3625 Started  Thu Nov 12 05:36:02 UTC 2020
Task 3625 Finished Thu Nov 12 05:36:03 UTC 2020
Task 3625 Duration 00:00:01
Task 3625 done

Succeeded
```

#### 1.1.2. Runtime Config 생성 

```
releases:
- name: dynatrace-oneagent
  version: 1.3.3

addons:
- name: dynatrace-oneagent-addon
  jobs:
  - name: dynatrace-oneagent
    release: dynatrace-oneagent
    properties:
      dynatrace:
        environmentid: <environmentId>
        apitoken: <paas-token>
        sslmode: all
        downloadurl: https://direct-download-url-for-agent
        hostgroup: example_hostgroupployment
        hosttags: landscape=production team=my_team
        hostprops: Department=Acceptance Stage=Sprint
        infraonly: 0
        validatedownload: false
        installerargs: USER=vcap GROUP=vcap
  include:
    deployments:
      - name-of-your-deployment
    stemcell:
      - os: ubuntu-xenial
  exclude:
    lifecycle: errand


# optional: extra addon configuration for Windows cells
- name: dynatrace-oneagent-windows-addon
  jobs:
  - name: dynatrace-oneagent-windows
    release: dynatrace-oneagent
    properties:
      dynatrace:
        environmentid: <environmentId>
        apitoken: <paas-token>
  include:
    deployments:
      - name-of-your-deployment
    stemcell:
      - os: windows1803
  exclude:
    lifecycle: errand
```

- 위 설정에서 infraonly: 0, 1 0일 경우 fullstack 1일 경우 infraonly mode로 변경 되어 모니터링 대상의 Metrics이 정해집니다.
- 위 설정에서 exclude, include를 통해 제외 할 대상의 VM을 선택합니다.

#### 1.1.3. Apply Change
- 해당 되는 VM에 agent를 설치 하기 위해 Apply Change를 실행합니다.


#### 1.1.4. 적용 확인
- Apply Change가 완료 되면 Dynatrace에 접속하여 올바르게 연동이 되었는지 확인합니다.
- Dynatrace에서 주로 확인하였던 수치는 아래와 같습니다.
 
	  1) VM의 생사
	
![prometheus1][dynatrace-image-1]

![prometheus2][dynatrace-image-2]

	2) Request & Response의 수행 시간
	
![prometheus3][dynatrace-image-3]

	3) Java Container의 JVM Memory
	
![prometheus4][dynatrace-image-4]




## 2. 운영 간 특이 사항
- TAS 내부의 Dynatrace에서 등록한 API에 주기적으로 ping을 시도하여 신규 버전이 있을 경우 자동으로 다운로드를 받고 적용 시키는 현상이 있었습니다. 하지만 Process를 재기동 하기전에는 신규 버전이 적용되지 않습니다.
- APM 대상의 Java Application들에게 Memory, CPU, Disk 사용률은 안정적이지만 원인 모를 http health endpoint에서 장애가 발생하는 의심 현상이 있습니다.
- 신규 Release 된 Oneagent를 적용 하였을 때 기존 System과(Go Lang 등)의 버전 차이로 인하여 장애가 발생 할 수 있음으로 기존에 테스트가 반드시 필요합니다.



[dynatrace-image-1]:./images/dynatrace-image-1.png
[dynatrace-image-2]:./images/dynatrace-image-2.png
[dynatrace-image-3]:./images/dynatrace-image-3.png
[dynatrace-image-4]:./images/dynatrace-image-4.png


