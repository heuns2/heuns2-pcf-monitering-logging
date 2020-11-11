# Tanzu Metrics Store - Prometheus 간 연동 방법

- 참고자료
	- [metrcis store release git](https://github.com/cloudfoundry/metric-store-release)
	- [grafana release git](https://github.com/vito/grafana-boshrelease)
	- [인증 설정 정보](https://www.bookstack.cn/read/grafana-v6.2/9263e66aea3a4371.md)
- 본 가이드는 bosh를 통해 설치 되지 않고 Local 또는 VM에 설치 되어 있는 Grafana에서 Pivotal Cloud Foundary metrics-store에서 Platform에서 구동 중인 VM과 App의 Metrics을 가져오는 방법에 대해서 설명 합니다.
- 연동 부분을 제외한 모든 설정은 default입니다.

## 1. Grafana Config  설정

### 1.1. defaults.ini 파일 설정
- [auth.generic_oauth] Block에 아래 라인을 설정합니다.

```
name = UAA
enabled = true
allow_sign_up = true
client_id = grafana
client_secret = eexj5xawrml9laroh7z2
scopes = openid uaa.resource doppler.firehose logs.admin cloud_controller.read
auth_url = https://login.sys.{DOMAIN}/oauth/authorize
token_url = https://login.sys.{DOMAIN}/oauth/token
api_url = https://login.sys.{DOMAIN}/userinfo
```
- [users] Block에 아래 라인을 설정합니다.

```
auto_assign_org_role = Editor
```

### 1.2. datasources 파일 설정
- datasource는 UI 또는 config파일을 통하여 아래 정보를 추가합니다.

```
apiVersion: 1
datasources:
 - name: Metric Store
   type: prometheus
   orgId: 1
   url: https://metric-store.sys.{SYSTEM_DOMAIN}.kr
   jsonData:
      httpMethod: "GET"
      keepCookies: []
      oauthPassThru: true
   editable: false
```


## 2. TAS UAA Client  설정
- Grafana에서 TAS의 Logging Architecture & metrics-store API 쿼리를 이용하기 위해 적절한 권한이 있는 TAS 내부 uaa client를 생성합니다.

```
$ uaac client add grafana \
--name grafana \
--secret eexj5xawrml9laroh7z2 \
--authorized_grant_types authorization_code,refresh_token \
--authorities uaa.none \
--scope openid,uaa.resource,doppler.firehose,logs.admin,cloud_controller.read \
--redirect_uri http://172.30.164.155:3000/login/generic_oauth
```

- 이때 사용되는 REDIRECT URI는 Grafana의 URI로 UAA의 인증을 통과한 다음의 위치입니다.

## 3. 연동 확인 사항
- Grafana Server를 재기동하여 변경사항을 적용합니다.
- 연동된 UAA Login 버튼을 통해 로그인하여 OAuth 인증을 실행합니다.
- Datasource에서 생성 된  Metric Store에서 save & test 버튼을 실행 및 prometheus dashboard에서 연동이 잘되었는지 확인 합니다.
- Datasource이 dashboard를 생성하고, 패널을 생성하여 아래 쿼리를 추가하여 정상 동작하는지 확인합니다.

```
system_mem_percent{job="metric-store", origin="bosh-system-metrics-forwarder", product="VMware Tanzu Application Service", source_id="bosh-system-metrics-forwarder", unit="Percent"}
```
