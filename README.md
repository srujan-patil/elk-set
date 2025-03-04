# ELK Stack Setup Documentation

## Overview
This repository contains setup instructions for deploying the **ELK Stack (Elasticsearch, Logstash, Kibana)** to collect, analyze, and visualize logs.

## Prerequisites
- Ubuntu Server (20.04/22.04)
- Minimum 4GB RAM (Recommended: 8GB+)
- Root or sudo access
- Open ports: `9200` (Elasticsearch), `5601` (Kibana), `5044` (Logstash)

## Installation Steps

### 1. Install Java (Required for Elasticsearch & Logstash)
```bash
sudo apt update
sudo apt install -y openjdk-11-jdk
java -version
```

### 2. Install Elasticsearch
```bash
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
echo "deb https://artifacts.elastic.co/packages/7.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-7.x.list
sudo apt update && sudo apt install -y elasticsearch
```
- Configure Elasticsearch:
```bash
echo "xpack.security.enabled: true" | sudo tee -a /etc/elasticsearch/elasticsearch.yml
```
- Start & Enable Elasticsearch:
```bash
sudo systemctl enable --now elasticsearch
```

### 3. Secure Elasticsearch (Set Up Authentication)
```bash
sudo /usr/share/elasticsearch/bin/elasticsearch-setup-passwords interactive
```

### 4. Install Kibana
```bash
sudo apt install -y kibana
```
- Configure Kibana:
```bash
echo "elasticsearch.username: 'kibana_system'" | sudo tee -a /etc/kibana/kibana.yml
echo "elasticsearch.password: '<YOUR_PASSWORD>'" | sudo tee -a /etc/kibana/kibana.yml
```
- Start & Enable Kibana:
```bash
sudo systemctl enable --now kibana
```

### 5. Install Logstash
```bash
sudo apt install -y logstash
```
- Configure Logstash for Filebeat Input:
```bash
sudo tee /etc/logstash/conf.d/filebeat.conf <<EOF
input {
  beats {
    port => 5044
  }
}
output {
  elasticsearch {
    hosts => ["http://localhost:9200"]
    index => "filebeat-%{+YYYY.MM.dd}"
  }
}
EOF
```
- Start & Enable Logstash:
```bash
sudo systemctl enable --now logstash
```

### 6. Install Filebeat (Log Forwarder)
```bash
sudo apt install -y filebeat
```
- Configure Filebeat:
```bash
sudo filebeat modules enable nginx apache
sudo filebeat setup -e
```
- Start & Enable Filebeat:
```bash
sudo systemctl enable --now filebeat
```

### 7. Verify ELK Setup
- Check Elasticsearch Indices:
```bash
curl -X GET "http://localhost:9200/_cat/indices?v"
```
- View Logs in Kibana:
  - Open `http://<your-server-ip>:5601`
  - Go to **Discover** → Select `filebeat-*`

## Advanced Configurations
### Secure ELK with Nginx Reverse Proxy
```bash
sudo apt install -y nginx
sudo htpasswd -c /etc/nginx/.kibana-user admin
```
- Configure Nginx Proxy (`/etc/nginx/sites-available/kibana`):
```nginx
server {
    listen 80;
    server_name your-domain.com;

    auth_basic "Restricted Access";
    auth_basic_user_file /etc/nginx/.kibana-user;

    location / {
        proxy_pass http://localhost:5601;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```
- Enable & Restart Nginx:
```bash
sudo ln -s /etc/nginx/sites-available/kibana /etc/nginx/sites-enabled/
sudo systemctl restart nginx
```

## Scaling ELK for Production
### Optimize Elasticsearch
- Increase JVM Heap Size (`/etc/elasticsearch/jvm.options`):
```bash
-Xms4g
-Xmx4g
```
- Configure Index Lifecycle Management (ILM):
```bash
curl -X PUT "localhost:9200/_ilm/policy/filebeat_policy" -H 'Content-Type: application/json' -d'
{
  "policy": {
    "phases": {
      "hot":    { "actions": { "rollover": { "max_age": "7d" } } },
      "delete": { "min_age": "30d", "actions": { "delete": {} } }
    }
  }
}'
```

## Troubleshooting
### Check Service Status
```bash
sudo systemctl status elasticsearch kibana logstash filebeat
```
### View Logs
```bash
sudo journalctl -u elasticsearch --no-pager | tail -n 20
sudo journalctl -u kibana --no-pager | tail -n 20
sudo journalctl -u logstash --no-pager | tail -n 20
sudo journalctl -u filebeat --no-pager | tail -n 20
```

## Conclusion
This guide provides a complete ELK setup for centralized logging and visualization. You can further enhance it by adding security features, scaling Elasticsearch, and integrating additional log sources.

---

## Author
Maintained by **[Your Name]**  
📧 Contact: your-email@example.com
