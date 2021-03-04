
1) Event Alert 설치
- Event Alert에서 Mysql 사용 명을 db-medium로 명시

2) "504 5.7.4 Unrecognized authentication type" 에러로 인하여 eva target 생성 중 장애 발생
- Event Alert Tile의 SMTP Config 화면에서 Enable TLS: False로 설정하여 실행

3) target 생성하고 leedh@gmail.com에서 알림 정책 확인을 클릭
3-1) Email 얀동
$ cf eva-create-target email-User-1 email leedh@gmail.com
3-2) Slack 연동
$ cf eva-create-target slack-User-2 slack https://hooks.slack.com/services/TMA9G12TZ/B01KSGR

4) 확인이 가능한 모든 topics 확인
$ cf eva-topics

5) 특정 topics를 구독
$ cf eva-subscribe email-User-1 healthwatch --topics healthwatch.diego.availablefreechunks,healthwatch.diego.availablefreechunksdisk

6) 특정 topics를 구독 해제
$ cf eva-unsubscribe email-User-1 healthwatch --topics healthwatch.diego.availablefreechunks,healthwatch.diego.availablefreechunksdisk

7) 현재 구독 중인 Topics 확인
$ cf eva-subscriptions
Getting subscriptions as tasadmin...
OK
Target         Publisher     Topics                                      Exclusions
email-User-1   healthwatch   healthwatch.diego.availablefreechunks       -
~              ~             healthwatch.diego.availablefreechunksdisk   -
slack-User-1   healthwatch   healthwatch.diego.availablefreechunks       -
~              ~             healthwatch.diego.availablefreechunksdisk   -

8) health watch 임계치 확인 & 상세 변경치 Sample
$ uaac target uaa.sys.leedh.cloud
Target: https://uaa.sys.leedh.cloud
Context: admin, from client admin

$ uaac token client get
Client ID:  healthwatch_api_admin
Client secret:  ********************************

Successfully fetched token via client credentials grant.
Target: https://uaa.sys.leedh.cloud
Context: healthwatch_api_admin, from client healthwatch_api_admin

$ uaac curl https://healthwatch-api.sys.leedh.cloud/v1/alert-configurations -k  > default
$ uaac curl -k "https://healthwatch-api.sys.leedh.cloud/v1/alert-configurations?q=origin == 'healthwatch' and name == 'Diego.AvailableFreeChunksDisk'"
$ uaac curl -k "https://healthwatch-api.sys.leedh.cloud/v1/alert-configurations?q=origin == 'gorouter' and name == 'total_routes'"
$ uaac curl -X POST "https://healthwatch-api.sys.leedh.cloud/v1/alert-configurations"  \
       -H "Content-Type: application/json" \
       --data "{\"query\":\"origin == 'healthwatch' and name == 'health.check.CanaryApp.available'\",\"threshold\":{\"critical\":0.5,\"warning\":0.5,\"type\":\"LOWER\"}}"ㄷ
