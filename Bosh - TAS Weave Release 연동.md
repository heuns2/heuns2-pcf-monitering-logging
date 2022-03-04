# Bosh - TAS Weave Release 연동
- Weave Scope는 Application Map을 자동으로 생성하므로 Container화 된 Micro Service 기반 Application을 직관적으로 이해, 모니터링 및 제어 할 수 있습니다.
- 본 문서는 Weave Scope App 화면과 Runtime Config를 통해 VM에 scope-probe, scope_garden 설치, Cloud Foundry에서 방출 되는 Metrics을 수집하기 위한 권한 정책 정립에 대한 방법을 설명 합니다.
- 사전 조건
    - 4040 PORT 방화벽 Open
    - 외부 환경이 되지 않는다면 Release Download 후 Bosh 명령어 장소로 이동
- 참고 링크
    - https://www.weave.works/docs/scope/latest/introducing/ 
- 고려 사항
    -  Application의 구조 (Start CLI, Cpu, Disk)등을 확인하기 위하여 "/v2/apps?inline-relations-depth=2&order-direction=asc&page=2&results-per-page=50"와 같은 Endpoint가 사용 된다, 이 Query는 Cloud Controller의 부하가 발생 할 수 있으며 Router의 Latency 증가의 원인이 될 염려가 있습니다.  

## 1. Bosh Release Upload
   
 - Bosh Director에 Weave Release를 Upload 합니다.  

```
# local에 weave download
$ wget https://github.com/heuns2/bosh-weave-scope/releases/download/0.0.17/weave-scope-v0.0.17.tgz

# bosh director에 upload
$ bosh upload-release ./weave-scope-v0.0.17.tgz

# upload 된 release 확인
$ bosh releases | grep weave
weave-scope                     0.0.17*         f0cc5de2+
```

## 2. wep app UI VM 배포

### 2.1. wep app manifest 편집

```
$ cat manifests/scope-app.yml
---
name: weave-scope
releases:
- name: weave-scope
  version: latest
stemcells:
- alias: default
  os: ubuntu-xenial
  version: latest
instance_groups:
- name: scope_app
  azs:
  - <cloud config의 AZs name으로 변경 필요>
  instances: 1
  jobs:
  - name: scope_app
    release: weave-scope
    consumes: {}
    provides:
      weave_scope_app:
        as: weave_scope_app
        shared: true
  vm_type: <cloud config의 vm type name으로 변경 필요>
  stemcell: default
  properties: {}
  update:
    serial: true
    max_in_flight: 1
  networks:
  - default:
    - dns
    - gateway
    name: <cloud config의 network name으로 변경 필요>
  persistent_disk_type: '<cloud config의 disk type name으로 변경 필요>'
  vm_extensions: [<cloud config의 lb 설정 name으로 변경 필요>]  
update:
  canaries: 1
  canary_watch_time: 30000-300000
  update_watch_time: 30000-300000
  max_in_flight: 1
  max_errors: 2
  serial: true

```

### 2.2. wep app VM 배포
- 편집 완료한 manifest를 기반으로 VM을 생성합니다.

```
$ bosh -d weave-scope deploy manifest.yml

Using environment '10.0.2.11' as client 'ops_manager'  

Task 2179

Task 2179 | 11:53:01 | Preparing deployment: Preparing deployment (00:00:02)
Task 2179 | 11:53:03 | Preparing deployment: Rendering templates (00:00:01)
Task 2179 | 11:53:04 | Preparing package compilation: Finding packages to compile (00:00:00)
Task 2179 | 11:53:04 | Updating instance scope_app: scope_app/8776de61-eca4-4e99-b4ab-66c6238842be (0) (canary)
Task 2179 | 11:53:05 | L executing pre-stop: scope_app/8776de61-eca4-4e99-b4ab-66c6238842be (0) (canary)
Task 2179 | 11:53:05 | L executing drain: scope_app/8776de61-eca4-4e99-b4ab-66c6238842be (0) (canary)
Task 2179 | 11:53:06 | L stopping jobs: scope_app/8776de61-eca4-4e99-b4ab-66c6238842be (0) (canary)
Task 2179 | 11:53:11 | L executing post-stop: scope_app/8776de61-eca4-4e99-b4ab-66c6238842be (0) (canary)
Task 2179 | 11:53:35 | L installing packages: scope_app/8776de61-eca4-4e99-b4ab-66c6238842be (0) (canary)
Task 2179 | 11:53:37 | L configuring jobs: scope_app/8776de61-eca4-4e99-b4ab-66c6238842be (0) (canary)
Task 2179 | 11:53:37 | L executing pre-start: scope_app/8776de61-eca4-4e99-b4ab-66c6238842be (0) (canary)
Task 2179 | 11:53:38 | L starting jobs: scope_app/8776de61-eca4-4e99-b4ab-66c6238842be (0) (canary)
Task 2179 | 11:54:09 | L executing post-start: scope_app/8776de61-eca4-4e99-b4ab-66c6238842be (0) (canary) (00:01:06)

Task 2179 Started  Thu Jun 10 11:53:01 UTC 2021
Task 2179 Finished Thu Jun 10 11:54:10 UTC 2021
Task 2179 Duration 00:01:09
Task 2179 done

Succeeded
```

### 2.3. wep app VM 확인
- bosh vms로 VM 형상 확인 및 URI Port 4040 확인

```
$ bosh -d weave-scope vms
Using environment '10.0.2.11' as client 'ops_manager'

Task 2688

. Done

Deployment 'weave-scope'

Instance                                        Process State  AZ               IPs        VM CID               VM Type    Active  Stemcell
scope_app/8776de61-eca4-4e99-b4ab-66c6238842be  running        ap-northeast-1a  10.0.4.60  i-055a002908a695558  t3.medium  true    bosh-aws-xen-hvm-ubuntu-xenial-go_agent/621.117

1 vms

Succeeded

$ curl http://10.0.4.60:4040
!doctype html>
<html class="no-js">
  <head>
    <meta charset="utf-8">
    <title>Weave Scope</title>
    <meta name="description" content="">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <script language="javascript">window.__WEAVEWORKS_CSRF_TOKEN = "$__CSRF_TOKEN_PLACEHOLDER__";</script>
  <script>window.__WEAVE_SCOPE_THEMES = JSON.parse('{"normal":"style-app-202a880dddb1e91553fd.css?60221ddd3530ad065107","contrast":"style-contrast-theme-7eebf591a8aa3d6ac179.css?60221ddd3530ad065107","publicPath":""}')</script>
  <link href="style-app-202a880dddb1e91553fd.css?60221ddd3530ad065107" rel="stylesheet"></head>
  <body>
    <!--[if lt IE 10]>
      <p class="browsehappy">You are using an <strong>outdated</strong> browser. Please <a href="http://browsehappy.com/">upgrade your browser</a> to improve your experience.</p>
    <![endif]-->
    <div class="wrap">
      <div id="app"></div>
    </div>
  <script type="text/javascript" src="vendors.js?60221ddd3530ad065107"></script><script type="text/javascript" src="app-202a880dddb1e91553fd.js?60221ddd3530ad065107"></script></body>
</html>
```

## 3. 권한 인증 설정 및 wep app runtime config 적용  

### 3.1. uaac 인증 설정
- weave_scope_app VM이 사용 할 client 인증을 생성합니다.

```
$ uaac target uaa.sys.leedh.cloud
$ uaac token client get admin -s <admin_client_secret>
$ uaac client add scope-admin   --name scope-admin    --secret scope-admin   --authorized_grant_types client_credentials,refresh_token   --authorities cloud_controller.admin
```

### 3.2. wep app runtime config 편집
- cloud foundry VM에 적용 할 runtime config를 편집 합니다.

```
$ cat manifests/runtime-config.yml
---
releases:
- name: weave-scope
  version: 0.0.17

addons:
- name: scope-probe
  jobs:
  - name: scope_probe
    release: weave-scope
    consumes:
      weave_scope_app:
        from: weave_scope_app
        deployment: weave-scope
- name: scope-garden
  include:
    jobs:
    - name: garden
      release: garden-runc
  jobs:
  - name: scope_garden
    release: weave-scope
    properties:
      weave:
        scope:
          probe:
            garden:
              addr: /var/vcap/data/garden/garden.sock
            cf:
              api_url: <api.{SYSTEM_DOMAIN으로 변경 필요}>
              client_id: scope-admin
              client_secret: scope-admin
              skip_ssl_verify: true
```

### 3.3. wep app runtime config upload
- bosh director에 runtime config를 업로드 합니다.

```
# runtime config upload
$ bosh update-runtime-config runtime-config.yml --name=weave

# runtime config 확인
$ bosh config --name weave --type runtime
Using environment '10.0.2.11' as client 'ops_manager'

ID          35
Type        runtime
Name        weave
Created At  2021-06-10 13:57:14 UTC
Content     ---
            addons:
            - jobs:
              - consumes:
                  weave_scope_app:
                    deployment: weave-scope
                    from: weave_scope_app
                name: scope_probe
                release: weave-scope
              name: scope-probe
            - include:
                jobs:
                - name: garden
                  release: garden-runc
              jobs:
              - name: scope_garden
                properties:
                  weave:
                    scope:
                      probe:
                        cf:
                          api_url: https://api.sys.leedh.cloud
                          client_id: scope-admin
                          client_secret: scope-admin
                          skip_ssl_verify: true
                        garden:
                          addr: "/var/vcap/data/garden/garden.sock"
                release: weave-scope
              name: scope-garden
            releases:
            - name: weave-scope
              version: 0.0.17
1 config
Succeeded

```

### 3.4. Apply Change

- TAS를 Apply Change 합니다.

## 4. UI 확인

- CONTAINERS

![weave-1][weave-1]

[weave-1]:./images/weave-image-1.PNG


- HOSTS

![weave-2][weave-2]

[weave-2]:./images/weave-image-2.PNG
