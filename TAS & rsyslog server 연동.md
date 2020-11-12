
PCF On ELK, Splunk를 통해 Logging을 관리하면 편리하긴 하지만 과도한 리소스 사용과 관리를 해야 할 필요가 있다고 생각하는것 같다.

# TAS Platform과 Rsyslog 연동
- 사전 준비 사항
	- rsyslog install
		- sudo apt-get install rsyslog
		- sudo yum install rsyslog

## 1. Rsyslog의 서버 구동
- 기본적으로 rsyslog를 설치하게 되면 자동으로 start 되어 있는 상태지만 config 파일 수정 시 reload가 필요하다.
- rsyslog의 기본 cli는 아래와 같다.
	- sudo service rsyslog start -> rsyslog server 실행
	- sudo service rsyslog stop -> rsyslog server 중지
	- sudo service rsyslog restart -> rsyslog server 재실행
	- sudo service rsyslog stauts -> rsyslog server 상태 확인

## 2. Rsyslog Config File 설정
- rsyslog config 파일의 default 위치는 "/etc/rsyslog.conf"에 존재 한다.
- 추가 및 주석 해제한 Properties는 아래와 같다.

```
$ModLoad imtcp # 로깅의 종류 선택
$InputTCPServerRun 514 # TCP Protocal을 사용하며 514번으로 syslog를 수집, 추 후 Tile에 대한 syslog config port

$template TmplMsg,"/PCF/FOUNDATION/messages_%app-name%_%$YEAR%-%$MONTH%-%$DAY%.log"

*.info;mail.none;authpriv.none;cron.none ?TmplMsg # 일반 Message, Auth Message, Cron Message는 해당 template으로 받지 않겠다고 선언
```

- 고려 대상의 template property 목록 (template property는 rsyslog $template를 만들 때 필요한 매개 변수이며 필터와 같은 역활을 할 수 있는 것 같다. ex: message에 if then else if 등 특정 문자열이 포함 된 경우), APP-NAME이 가장 쓸만한것같다.
	- HOSTNAME: PCf Platform에서 rsyslog 쪽으로 Log를 보내는 Client의 IP가 나옴
	- SYSLOGSERVERITY: Log의 Info, warning, error Level 확인
	- APP-NAME: Log 추출 대상의 Name 확인, PCF Platform에서는 컴포넌트의 Job Name, App Guid가 추출 됨
	- PROGRAMNAME: app_name 뒤에 info추가
	- PROCID: procid가 표시 됨, rc로 시작되는 문자열이 추가 됨
	- SYSLOGTAG: PCF Platform이 보내는 syslog Tag가 나옴, app-name 뒤에 procid가 붙는다.


## 3. Rsyslog Config 추가 설정(확인 필요)

### 3.1. Rsyslog Log Rotation
- Rsyslog Config File 설정

```
$outchannel log_rotation,/var/log/log_rotation.log, 52428800,/home/test/./log_rotation_script
```

- script 작성

```
mv -f /var/log/log_rotation.log /var/log/log_rotation.log.1
```
