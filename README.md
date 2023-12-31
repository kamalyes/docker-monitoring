### 安装 node export

第一步：下载进入官网 [https://prometheus.io/download/#node\_exporter](https://prometheus.io/download/#node_exporter)  下载之后解压缩

```bash
# 我这里直接下载最新-0.17.0版本的包了
[root@k8s-master ~]# wget -O node_exporter.tar.gz https://github.com/prometheus/node_exporter/releases/download/v1.7.0/node_exporter-1.7.0.linux-amd64.tar.gz
[root@k8s-master ~]# tar xvzf node_exporter-1.7.0.linux-amd64.tar.gz && mv node_exporter-1.7.0.linux-amd64/ /usr/local/node_exporter/ && cd /usr/local/node_exporter
```

或使用以下脚本直接运行也可

```bash
[root@k8s-master ~]# tee install_node_exporter.sh <<-'EOF'
#!/bin/bash
groupadd -r prometheus
useradd -r -g prometheus -s /sbin/nologin -M -c "prometheus Daemons" prometheus
wget https://github.com/prometheus/node_exporter/releases/download/v1.7.0/node_exporter-1.7.0.linux-amd64.tar.gz
tar xvf node_exporter-1.7.0.darwin-amd64.tar.gz
mv node_exporter-1.7.0.darwin-amd64/ /usr/local/node_exporter/
chown -R prometheus.prometheus /usr/local/node_exporter/
cat <<END> /usr/lib/systemd/system/node_exporter.service
[Unit]
Description=node_exporter
Documentation=https://github.com/prometheus/node_exporter
After=network.target

[Service]
User=prometheus
Group=prometheus
ExecStart=/usr/local/node_exporter/node_exporter --web.config.file=/usr/local/node_exporter/config.yml --web.listen-address=:29100 --collector.systemd --collector.systemd.unit-whitelist=(sshd|nginx).service --collector.processes --collector.tcpstat
Restart=on-failure

[Install]
WantedBy=multi-user.target
END
chmod +x install_node_exporter.sh
sh install_node_exporter.sh
chmod o+w /etc/systemd/system/node_exporter.service
chmod o+w /usr/local/node_exporter/node_exporter
```

第二步：基于Basic Auth 配置Prometheus访问用户名&密码

```bash
# 没有https-tools则 yum install -y httpd-tools
[root@k8s-master node_exporter]# rpm -qa|grep httpd-tools   
httpd-tools-2.4.6-97.el7.centos.x86_64
[root@k8s-master node_exporter]# htpasswd -nBC 12 '' | tr -d ':\n'
New password:    # 这里设置密码为12356789
Re-type new password: 
$2y$12$YIKlmXMfavvrR2nNoD/2beFqOm7NSucdq7FZQoeAoAFH3LnH471.C
[root@k8s-master node_exporter]# openssl req -new -newkey rsa:2048 -days 365 -nodes -x509 -keyout node_exporter.key -out node_exporter.crt -subj "/C=CN/ST=Beijing/L=Beijing/O=Moelove.info/CN=localhost"
```

在node-exporter`config.yml`中新增配置文件

```bash
[root@k8s-master node_exporter]# cat config.yml 
basic_auth_users:
  prometheus: '$2y$12$YIKlmXMfavvrR2nNoD/2beFqOm7NSucdq7FZQoeAoAFH3LnH471.C' # 12356789 (prometheus:用户名、后面会用到)
tls_server_config:
  cert_file: node_exporter.crt
  key_file: node_exporter.key
```

最后：启动并设置开机自启

```bash
systemctl daemon-reload
systemctl enable node_exporter
systemctl start node_exporter
systemctl status node_exporter.service
```

### 安装prometheus、grafana

```bash
[root@k8s-master ~]# mkdir -p /opt/docker-monitoring/
[root@k8s-master ~]# cp ../docker-monitoring/ /opt/docker-monitoring/ && chmod 777 -R /opt/docker-monitoring/ && cd /opt/docker-monitoring/
[root@k8s-master docker-monitoring]# rz node_exporter.crt、node_exporter.key放到prometheus/ssl目录下
[root@k8s-master docker-monitoring]# docker-compose up -d
[root@k8s-master docker-monitoring]# vi prometheus/prometheus.yml #将下面内容修改好即可
scrape_configs:
   # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prod-node-exporter'
    scheme: https
    tls_config:
      cert_file: /etc/prometheus/ssl/node_exporter.crt
      key_file: /etc/prometheus/ssl/node_exporter.key
      server_name: "prod"
      insecure_skip_verify: true
    basic_auth:
      username: prometheus
      password: 12356789
    static_configs:
     #监听的地址
     - targets: ['${PROD_NODE_EXPORT_IP_1}:${PORT}','${PROD_NODE_EXPORT_IP_2}:{PORT}']
```
