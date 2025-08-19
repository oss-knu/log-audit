# Teleport + Logstash + OpenSearch 로그 수집 가이드

Teleport에서 발생하는 **Audit Logs**를 Logstash를 통해 OpenSearch에 저장하고, OpenSearch Dashboard에서 확인 및 필터링하는 부분.

</br></br>

## 🏗️ 아키텍처 개요

```
Teleport → Logstash → OpenSearch → OpenSearch Dashboard
```

* **Teleport**: 인증/접속/세션 이벤트 로그 생성
* **Logstash**: 로그 수집 및 OpenSearch 전송
* **OpenSearch**: 로그 저장 및 검색
* **OpenSearch Dashboard**: 로그 시각화 및 필터링


</br></br>

## ⚙️ Teleport 설정 (teleport.yaml)

Teleport에서 Audit 로그를 **로컬 파일**로 저장하도록 설정합니다.

```yaml
version: v3
teleport:
  nodename: root-cluster
  data_dir: /var/lib/teleport
  log:
    output: data/log/teleport.log # 로그를 지정된 위치에 저장.
    severity: INFO
    format:
      output: json # json으로 formatting함.

auth_service:
  enabled: true
  cluster_name: root-cluster-test

proxy_service:
  enabled: true

ssh_service:
  enabled: true

audit_service:
  enabled: true
  audit_events_uri:
    - file:/var/lib/teleport/logs/teleport.log
```

</br></br>

## 📦 Logstash 설정 (Pipeline 역할)


> Logstash가 Teleport-Opensearch를 연결하는 **Pipeline 역할**을 함.


### input: Teleport 로그 파일

```ruby
input {
  file {
    path => "../../data/logs/teleport.log" # 로컬 파일을 읽음.
    codec => json
    start_position => "beginning"
    sincedb_path => "/dev/null"
  }
}
```

### output: OpenSearch를 설정.

```ruby
output {
  opensearch {
    hosts => ["https://opensearch:9200"]
    user => "logstash"
    password => "your_password"
    index => "audit-events-%{+YYYY.MM.dd}" # index를 지정하여 보냄.
    ssl => true
  }
}
```

</br></br>

## 🔎 OpenSearch Dashboard에서 로그 확인

### 인덱스 패턴 생성

* `audit-events-*` 패턴으로 생성
* Discover 메뉴에서 최근 로그 확인

### 필드 기반 필터

* `event_type: "login"`
* `event_type: "auth"`

쿼리 예시:

```json
GET audit-events-*/_search
{
  "query": {
    "term": {
      "event_type": "login"
    }
  }
}
```


