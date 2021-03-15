# 1. Tanzu Platform - Zabbix Agent 연동
- 본 가이드는 Zabbix Agent를 Bosh Release 형태로 작성하여 Runtime Config 형태로 Bosh Deployments에 배포하여 Zabbix Server UI에서 확인하는 가이드입니다.


## 1.1. Zabbix Release Git Clone
- Zabbix release Source Git Clone

```
$ git clone https://github.com/heuns2/bosh-zabbix-release.git
```

- Tree 구조
	- Jobs: Zabbix 실행 Config 파일과 서비스 실행 스크립트
	- Packages: Zabbix 실행 소스 코드 파일에 대한 버전 명시 및 소스 코드 Build
	- Blobs: Packages에서 사용 될 실제 소스 코드 명시

```
ubuntu@ubuntu:~/workspace/zabbix/bosh-zabbix-release$ tree
.
├── blobs
│   ├── pcre_8.43
│   │   └── pcre-8.43.tar.gz
│   └── zabbix_4.4.0
│       └── zabbix-4.4.0.tar.gz
├── config
│   ├── blobs.yml
│   └── final.yml
├── jobs
│   └── zabbix_agentd
│       ├── monit
│       ├── spec
│       └── templates
│           ├── ctl.erb
│           └── zabbix_agentd.conf.erb
├── manifests
│   ├── cloud-config
│   ├── manifest_old.yml
│   ├── manifest.yml
│   └── zabbix-runtime.yml
├── packages
│   ├── pcre_8.43
│   │   ├── packaging
│   │   └── spec
│   └── zabbix_agentd_4.4.0
│       ├── packaging
│       └── spec
├── README.md
└── src
```

## 1.2. Bosh Release Create & Upload
- 작성한 Script, Package 파일을 기반으로 bosh create-release 실행

```
# Bosh Release 패키징
$ bosh create-release --name zabbix_agentd --version 1.0 --tarball=zabbix-agentd-release.tgz
Adding job 'zabbix_agentd/82ad45bd23a400f25f66b5f77723ef52d876098b4b3ae3d55ffbc80f0d001ba0'...
Added job 'zabbix_agentd/82ad45bd23a400f25f66b5f77723ef52d876098b4b3ae3d55ffbc80f0d001ba0'
Adding package 'pcre_8.43/7e35cb65abcec3a7d9ba46125623eb307472fe1f6c6574a83f02a291ccc039c5'...
Added package 'pcre_8.43/7e35cb65abcec3a7d9ba46125623eb307472fe1f6c6574a83f02a291ccc039c5'
Adding package 'zabbix_agentd_4.4.0/ed804c4eef4839c7b2448d4c3063ebb1be4a8bf60734e5ce1e0d0ead2d94b204'...
Added package 'zabbix_agentd_4.4.0/ed804c4eef4839c7b2448d4c3063ebb1be4a8bf60734e5ce1e0d0ead2d94b204'

Added dev release 'zabbix_agentd/1.0'

Name         zabbix_agentd
Version      1.0
Commit Hash  e4ab16e
Archive      /home/ubuntu/workspace/zabbix/bosh-zabbix-release/zabbix-agentd-release.tgz

Job                                                                             Digest                                                                   Packages
zabbix_agentd/82ad45bd23a400f25f66b5f77723ef52d876098b4b3ae3d55ffbc80f0d001ba0  sha256:2295e9fd7b2826185430043251f1d83fc6c6302cbc3b11d189908c8d1b11d96c  zabbix_agentd_4.4.0

1 jobs

Package                                                                               Digest                                                                   Dependencies
pcre_8.43/7e35cb65abcec3a7d9ba46125623eb307472fe1f6c6574a83f02a291ccc039c5            sha256:fda07bcebc5062c6241bef0e4699dd9d9ca6b27f69bdda87a8820814a866cde9  -
zabbix_agentd_4.4.0/ed804c4eef4839c7b2448d4c3063ebb1be4a8bf60734e5ce1e0d0ead2d94b204  sha256:22b083bd65ba3b8167e1175488535c8306845838f3f3fc773035dba2eee0b344  pcre_8.43

2 packages

Succeeded


# Bosh Release Upload
$ bosh upload-release ./zabbix-agentd-release.tgz
######################################################## 100.00% 104.21 KiB/s 0s
Task 5269

Task 5269 | 05:09:06 | Extracting release: Extracting release (00:00:00)
Task 5269 | 05:09:06 | Verifying manifest: Verifying manifest (00:00:00)
Task 5269 | 05:09:06 | Resolving package dependencies: Resolving package dependencies (00:00:00)
Task 5269 | 05:09:06 | Creating new packages: pcre_8.43/7e35cb65abcec3a7d9ba46125623eb307472fe1f6c6574a83f02a291ccc039c5 (00:00:00)
Task 5269 | 05:09:06 | Creating new packages: zabbix_agentd_4.4.0/ed804c4eef4839c7b2448d4c3063ebb1be4a8bf60734e5ce1e0d0ead2d94b204 (00:00:01)
Task 5269 | 05:09:07 | Creating new jobs: zabbix_agentd/82ad45bd23a400f25f66b5f77723ef52d876098b4b3ae3d55ffbc80f0d001ba0 (00:00:00)
Task 5269 | 05:09:07 | Release has been created: zabbix_agentd/1.0 (00:00:00)

Task 5269 Started  Mon Mar 15 05:09:06 UTC 2021
Task 5269 Finished Mon Mar 15 05:09:07 UTC 2021
Task 5269 Duration 00:00:01
Task 5269 done

Succeeded
```


## 1.3. Zabbix manifest 작성 및 배포 & Update Runtime Config

- server_ip: 192.168.0.5는 Zabbix Server의 IP

```
$ cat manifests/zabbix-runtime.yml
addons:
- name: zabbix-agentd
  include:
    deployments:
    - appMetrics-43af9618b9d8fa50d087
  jobs:
  - name: zabbix_agentd
    release: zabbix-agentd
    properties:
      zabbix:
        server_ip: 192.168.0.5
releases:
- name: zabbix-agentd
  version: "1.0"

$ bosh update-runtime-config --name zabbix-agent zabbix-runtime.yml
```

## 1.4. Apply Change
- Zabbix 설치 대상의 Bosh Deployments을 Apply Change 합니다.
- 설치 된 VM에서 아래 정보를 확인 합니다.

```
$ cat /var/vcap/jobs/zabbix_agentd/
bin/      .bosh/    conf/     monit     packages/
log-store-vms/7d71c1ab-2fc9-4f9e-ae74-a9a7c7e0edd4:~$ cat /var/vcap/jobs/zabbix_agentd/conf/zabbix_agentd.conf
Server=192.168.0.5
ServerActive=192.168.0.5
Hostname=appMetrics-43af9618b9d8fa50d087.log-store-vms.0
HostMetadata=bosh
RefreshActiveChecks=300
SourceIP=0.0.0.0
PidFile=/var/vcap/sys/run/zabbix_agentd/zabbix_agentd.pid
LogFile=/var/vcap/sys/log/zabbix_agentd/zabbix_agentd.log
```

## 2.1. Zabbix Server Web UI 확인

- Zabbix Server Web UI에 접속하여 아래 정보를 설정 합니다, 이때 host 정보는 위 config에서 등록 된 appMetrics-43af9618b9d8fa50d087.log-store-vms.0 = deployment_name.job_name.index 입니다.

- Zabbix Server에서 Hosts를 생성합니다. 이때 hostname, ip는 runtime config를 통해 설치 된 VM의 IP입니다.

![zabbix-1][zabbix-1]

[zabbix-1]:./images/zabbix-image-1.PNG


![zabbix-2][zabbix-2]

[zabbix-2]:./images/zabbix-image-2.PNG

- Zabbix Server와 Zabbix Agent가 서로 통신이 되게 되면 초록색으로 활성화 표시가 나타납니다.

![zabbix-3][zabbix-3]

[zabbix-3]:./images/zabbix-image-3.PNG


- 일정 시간이 지나면 모니터링 화면에서 아래와 같은 Metrics 정보가 확인 됩니다.

![zabbix-4][zabbix-4]

[zabbix-4]:./images/zabbix-image-4.PNG




