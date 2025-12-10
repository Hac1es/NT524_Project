# Cài Đặt ELK Dashboard

---

## Step 1: Chuẩn bị môi trường trên máy giám sát (Monitor)

Cài đặt các gói cần thiết và cập nhật hệ thống:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install net-tools vim curl wget -y
```

Đổi tên host:

```bash
sudo hostnamectl set-hostname monitor
```


Thêm GPG Key và repository của ELK Stack:

```bash
sudo wget https://artifacts.elastic.co/GPG-KEY-elasticsearch -O /etc/apt/keyrings/GPG-KEY-elasticsearch.key
sudo echo "deb [signed-by=/etc/apt/keyrings/GPG-KEY-elasticsearch.key] https://artifacts.elastic.co/packages/8.x/apt stable main" | sudo tee /etc/apt/sources.list.d/elastic-8.x.list
sudo apt update
```

---

## Step 2: Cài đặt ELK Stack

### 2.1: Cài đặt Java

```bash
sudo apt install openjdk-17-jdk -y
java -version
```

### 2.2: Cài đặt Elasticsearch

```bash
sudo apt install elasticsearch -y
sudo systemctl daemon-reload
sudo systemctl start elasticsearch
sudo systemctl enable elasticsearch
sudo systemctl status elasticsearch
```

Config Elasticsearch:
```bash
sudo nano /etc/elasticsearch/elasticsearch.yml
```

```yaml
cluster.name: ossec_cluster
node.name: elasticsearch_node
network.host: 0.0.0.0
xpack.security.enabled: false
```

Lưu thay đổi & kiểm tra:
```bash
sudo systemctl restart elasticsearch
curl -X GET "localhost:9200"
```

### 2.3: Cài đặt Kibana

```bash
sudo apt install kibana -y
sudo systemctl start kibana
sudo systemctl enable kibana
sudo systemctl status kibana
```

Config Kibana:
```bash
sudo nano /etc/kibana/kibana.yml
```

```yaml
server.port: 5601
server.host: 0.0.0.0
elasticsearch.hosts: ["http://localhost:9200"]
```

```bash
sudo systemctl restart kibana
```

Truy cập Kibana: http://localhost:5601

### 2.4: Cài đặt Logstash

```bash
sudo apt install logstash -y
sudo systemctl start logstash
sudo systemctl enable logstash
sudo systemctl status logstash
```

---

## Step 3: Kết nối OSSEC Manager với máy Monitor

* Trên OSSEC Manager:
Chỉnh file ossec.conf:

```bash
sudo nano /var/ossec/etc/ossec.conf
```

```conf
# Thêm đoạn này vào file, trong block <ossec_config> (IP trong <server> là IP server)
<syslog_output>                 
  <server>192.168.56.12</server>                 
  <port>5001</port>                 
  <format>default</format>
</syslog_output>
```

Enable việc OSSEC gửi log:

```bash
sudo /var/ossec/bin/ossec-control enable client-syslog
```

Kiểm tra file config và khởi động:

```bash
sudo /var/ossec/bin/ossec-logtest -t
sudo /var/ossec/bin/ossec-control start
```

* Trên máy Monitor:

Thêm mới 1 file config cho logstash:
```bash
sudo nano /etc/logstash/conf.d/ossec-logstash.conf
```

```conf
input {
  udp {
    port => 5001
    type => "ossec"
  }
}

filter {
  if [type] == "ossec" {
    grok {
      match => { "message" => "%{SYSLOGTIMESTAMP:syslog_timestamp} %{SYSLOGHOST:syslog_host} %{DATA:syslog_program}: Alert Level: %{BASE10NUM:Alert_Level}; Rule: %{BASE10NUM:Rule} - %{GREEDYDATA:Description}; Location: %{GREEDYDATA:Details}" }
      add_field => [ "ossec_server", "%{host}" ]
    }
    mutate {
      remove_field => [ "syslog_hostname", "syslog_message", "syslog_pid", "message", "@version", "type", "host" ]
    }
  }
}

output {
  elasticsearch {
    hosts => ["http://localhost:9200"]
    index => "ossec-%{+YYYY.MM.dd}"
  }
}
```

Kiểm tra:
```bash
sudo -u logstash /usr/share/logstash/bin/logstash --path.settings /etc/logstash -t
```

Cuối cùng, khởi động lại logstash:
```bash
sudo systemctl restart logstash
```

---