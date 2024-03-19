您好，以下是我关于[运维问卷](https://github.com/housesigma/hr-interview/blob/main/DevOps.md)的回答。
> 由于篇幅原因，这里不再写完整的题目内容，只通过标题来区分
>
> 这些题目参考了ChatGPT

### 一、写一个定时执行的Bash脚本，每月的一号凌晨1点 对 MongoDB 中 test.user_logs 表进行备份、清理
> 这个脚本假设MongoDB 已经配置好，并且服务器上已经安装了必要的工具（如 mongodump 等）

> 这个脚本假设user_logs这个集合中的文档有一个叫created_at字段，数据类似于:
```bash
test> db.user_logs.find()
[
  {
    _id: ObjectId('65f9651b866ae34b1edb83b0'),
    user_id: 1,
    action: 'login',
    created_at: '2024-02-01T08:00:00.000Z'
  },
  {
    _id: ObjectId('65f9651b866ae34b1edb83b4'),
    user_id: 2,
    action: 'login',
    created_at: '2024-03-15T09:00:00.000Z'
  }
]
```
如下是备份脚本：backup_mongo_user_logs.sh
```bash
#!/bin/bash

# 获取上个月的年份和月份
last_month=$(date -d "last month" +%Y-%m)
this_month=$(date -d "this month" +%Y-%m)

# 设置备份目录和文件名
BACKUP_DIR="/data/mongo-bak/"
BACKUP_FILE="user_logs_${last_month}"

# 检测执行结果并触发 webhook
# 如果 webhook 可以接收参数的话，则可以在告警中知道是哪一步出错了，这里暂时没写
function check_status {
  if [ $? -ne 0 ]; then
    curl -X POST https://monitor.ipo.com/webhook/mongodb
    exit 1
  fi
}

# 备份 MongoDB 数据到本地
mongodump --db test --collection user_logs --out="${BACKUP_DIR}/${BACKUP_FILE}" --query "{ \"created_at\": { \"\$gte\": \"${last_month}-01T00:00:00.000Z\", \"\$lt\": \"${this_month}-01T00:00:00.000Z\" } }"
check_status

# 打包备份文件
tar -zcvf ${BACKUP_DIR}/${BACKUP_FILE}.tar.gz ${BACKUP_DIR}/${BACKUP_FILE}
check_status

# 使用 sftp 传输备份文件到远程服务器
# 这一步未测试
sftp bak@bak.ipo.com <<< $"put ${BACKUP_DIR}/${BACKUP_FILE}.tar.gz"
check_status

# 清理备份数据
mongosh <<EOF
use test
db.user_logs.deleteMany({
    "created_at": {
        "\$gte": "${last_month}-01T00:00:00.000Z",
        "\$lt": "${this_month}-01T00:00:00.000Z"
    }
});
EOF
check_status
```
将如下定时任务写入对应的执行用户的crontab中，例如root的文件路径是：/var/spool/cron/root
```
00 01 01 * * /bin/bash /path/to/backup_mongo_user_logs.sh
```

### 二、根据要求提供一份Nginx配置
> 未在服务器上详细测试，只是通过nginx -t执行了检查
```nginx
server {
    listen 80;
    server_name ipo.com;
    # 非HTTP请求经过301重定向到HTTPS
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name ipo.com;

    # SSL证书配置
    ssl_certificate /path/to/your/certificate.pem;
    ssl_certificate_key /path/to/your/private.key;

    # /api/{name}路径的请求通过unix sock发送到本地php-fpm
    location ~ ^/api/(.*)$ {
        # 限流设置，每秒1.5个请求
        limit_req zone=api_limit burst=1 nodelay;

        # FastCGI配置
        fastcgi_pass unix:/var/run/php/php-fpm.sock;
        fastcgi_param SCRIPT_FILENAME /www/api/$1.php;
        fastcgi_index index.php;
        include fastcgi_params;
    }

    # /static/目录下的静态文件优化
    location /static/ {
        alias /static/;
        expires max;
        add_header Cache-Control "public";
    }

    # 根据UA进行判断，反向代理到server_bot
    # 其他请求指向目录 /www/ipo/
    location / {
        if ($http_user_agent ~* "Google Bot") {
            proxy_pass http://bot.ipo.com;
            break;
        }

        root /www/ipo;
        try_files $uri $uri/ /index.html /public/index.html /api/landing;
    }

    # 限流区域设置
    limit_req_zone $binary_remote_addr zone=api_limit:10m rate=1.5r/s;
}
```
### 三、通过iptables进行Docker网络配置
```bash
Docker_A_IP="172.17.0.2"
Docker_B_IP="172.17.0.3"
Docker_B_IP="172.17.0.4"

# 允许内网IP范围访问所有容器
iptables -A INPUT -i docker0 -s 192.168.0.1/27 -j ACCEPT
iptables -A FORWARD -i docker0 -s 192.168.0.1/27 -j ACCEPT

# Docker_A 与 Docker_B 相互访问
iptables -A FORWARD -i docker0 -o docker0 -s {Docker_A_IP} -d {Docker_B_IP} -j ACCEPT
iptables -A FORWARD -i docker0 -o docker0 -s {Docker_B_IP} -d {Docker_A_IP} -j ACCEPT

# Docker_C 不能访问其它两个容器
iptables -A FORWARD -i docker0 -o docker0 -s {Docker_C_IP} -j DROP

# Docker_A:8080 对外提供服务
iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 8080 -j DNAT --to-destination {Docker_A_IP}:8080
iptables -t nat -A POSTROUTING -o docker0 -p tcp --dport 8080 -d {Docker_A_IP} -j SNAT --to-source {Docker_A_IP}

# Docker_C:80 对外提供服务
iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 80 -j DNAT --to-destination {Docker_C_IP}:80
iptables -t nat -A POSTROUTING -o docker0 -p tcp --dport 80 -d {Docker_C_IP} -j SNAT --to-source {Docker_C_IP}

# 保存规则，确保重启后生效
iptables-save > /etc/iptables/rules.v4
```

### 四、生产环境数据库主从master服务器切换

### 五、Haproxy代理的MySQL Slave集群，偶尔会产生 SQLSTATE[HY000]: General error: 2006 MySQL server has gone away 的错误，请根据经验，给出一排查方案与可能的方向
