## PAS Platform - ELK RPM Package 설치(Centos OS)
- 해당 문서는 PAS Platform과 ELK 간의 기본 연동 방법에 대해 설명 합니다.
- 해당 문서는 Redhat 기반으로 작성 되었습니다. 

### 1. ELK Package(rpm) 설치
- 다운로드 링크는 아래와 같습니다.
	- https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.6.0-x86_64.rpm
	- https://artifacts.elastic.co/downloads/logstash/logstash-7.6.0.rpm
	- https://artifacts.elastic.co/downloads/kibana/kibana-7.6.0-x86_64.rpm

#### 1.1. Logstash Package Configuration
- Logstash는 다양한 소스에서 로그를 수집 할 수 있으며 수집된 로그에 대한 가공 & 변환하며 다른 데이터 저장소(Elasticsearch, datadog, redmind)으로 데이터를 전송 할 수 있습니다.
- RPM으로 설치를 하게 되면 Logstash의 Configuration File은 /etc/logstash/ 폴더에 저장 됩니다.


```
$ rpm -i logstash-7.6.0.rpm
```

- Logstash의 기본 설정 파일은은 아래와 같습니다. /etc/logstash/conf.d/logstash.conf

```
input {
  tcp {
     port => 5514 # 수집 대상의 PORT, cf drain 시 사용 되는 PORT 입니다.
  }
}
output {
  stdout {
    codec => json_lines
  }
  elasticsearch {
    hosts => "localhost:9200" # Elastic Search 서버의 주소
    index => "logstash-%{+YYYY.MM.dd}" # Elastic Search에 저장 될 Index 명 추 후 Kibana에서 Index 검색이 가능합니다
  }
}
```


#### 1.2. Elasticsearch Package Configuration
- ElasticSearch는 RESTful API 기반의 검색 및 분석 엔진입니다. Logstash에 대한 데이터를 저장하고 정형 데이터, 비정형 데이터, 위치 정보, 메트릭 등 다양한 유형의 데이터를 사용자가 원하는 방식으로 검색하고 결합할 수 있도록 지원합니다
- RPM으로 설치를 하게 되면 Elasticsearch 의 Configuration File은 /etc/elasticsearch/ 폴더에 저장 됩니다.

```
$ rpm -i elasticsearch-7.6.0-x86_64.rpm
```

- elasticsearch의 Config 변경 아래 속성 값을 변경 합니다. /etc/elasticsearch/elasticsearch.yml

```
http.port: 9200 # 9200에서 변경 시 logstash, kibana의 정보도 변경이 필요 합니다.
```


#### 1.3. Kibana Package Configuration
- Elasticsearch 데이터를 시각화합니다.
- RPM으로 설치를 하게 되면 Kibana의 Configuration File은 /etc/kibana/ 폴더에 저장 됩니다.

```
rpm -i kibana-7.6.0-x86_64.rpm
```

- kibana의 Config 변경 아래 속성 값을 변경 합니다. /etc/kibana/kibana.yml


```
elasticsearch.hosts: "http://localhost:9200"
```

### 2. ELK Package(rpm)  실행
- ELK 기본적인 Configuration을 완료하고 Service를 시작합니다.
- rpm으로 설치 시 자동으로 linux service, systemctl에 등록 됩니다.
- ELK Package Service 관리 명령어는 아래와 같습니다.

```
# logstash
$ service logstash start
$ service logstash stop
$ service logstash status
$ service logstash restart

# elasticsearch
$ service elasticsearch start
$ service elasticsearch stop
$ service elasticsearch status
$ service elasticsearch restart

# kibana
$ service kibana start
$ service kibana stop
$ service kibana status
$ service kibana restart
```


### 3. Drain Space
- Logstash의 IP:PORT를 통해 cf drain 명령어를 실행 합니다.
- 실행 예시

```
$ cf drain-space syslog://{LOG_STASH_IP}:5514 --drain-name=common-drain --type logs
```

### 4. 기타
- 현재의 Logstash Default 설정일 경우 많은 양의 Log가 쌓일 때 확인이 어렵고 하나의 Index에 쏠림 현상이 있습니다. 이때 PAS에서 수집되는 {ORG}-{SPACE}로 필터로 변경하려면 Logstash Config 파일 수정이 필요 합니다.

```
input {
  tcp {
     port => 5514
  }
}
filter{
 grok {
    match => { "message" => "%{SYSLOG5424PRI}%{NONNEGINT:syslog5424_ver} +(?:%{TIMESTAMP_ISO8601:syslog5424_ts}|-) +(?:%{HOSTNAME:syslog5424_host}|-) +(?:%{NOTSPACE:syslog5424_app}|-) +(?:%{NOTSPACE:syslog5424_proc}|-) +(?:%{WORD:syslog5424_msgid}|-) +(?:%{SYSLOG5424SD:syslog5424_sd}|-|)%{SPACE}%{GREEDYDATA:message}" }
    add_tag => [ "CF", "CF-%{syslog5424_proc}", "parsed"]
    add_field => { "format" => "cf" }
    tag_on_failure => [ ]
    overwrite => [ "message" ]
  }
  mutate {
        split => ["syslog5424_host", "."]
        add_field => { "cf-org" => "%{[syslog5424_host][0]}" }
        add_field => { "cf-space" => "%{[syslog5424_host][1]}" }
        add_field => { "cf-app" => "%{[syslog5424_host][2]}" }
  }
}
output {
  stdout {
    codec => json_lines
  }
  elasticsearch {
    hosts => "localhost:9200"
    #index => "logstash-%{+YYYY.MM.dd}"
    #index => "%{[@cf-org][cf-space]}"
    index => "%{[cf-org]}-%{[cf-space]}-%{+YYYY.MM.dd}"
  }
}
```

#### 4.1. Logstash filter 부분 주요 설정 설명
- 아래 로그 중 필요 한 정보 test.common.test-common-space의 위치를 파악합니다.

```
1429 <14>1 2020-02-19T07:23:39.577777+00:00 test.common.test-common-space da21ab19-d8d0-4347-8256-1280c5bf7e72 
[RTR/2] - [tags@47450 __v1_type="LogMessage" app_id="da21ab19-d8d0-4347-8256-
...생략
```
- grok를 통하여 message의 주요 설정을 각 각의 변수로 담습니다. syslog5424_host에 test.common.test-common-space를 담습니다.
- syslog5424_host를 .으로 split하여 org, space, app 명을 새로운 add_field하여 추가합니다.
- output  설정에 ELK 설정 부분에 add_field한 filed 명으로 Index를 걸어주고 Logstash Service를 재실행 합니다.

- Kibana 화면에서의 ElasticSearch 결과물

```
test-common-2020.02.20 yellow open 1 1 17 24kb
test-back-2020.02.20 yellow open 1 1 10 33.9kb
```

확인 URL: http://{LOG_STASH_IP}:5601/app/kibana#/home?_g=()
