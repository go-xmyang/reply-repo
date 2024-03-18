您好，以下是我关于[运维问卷](https://github.com/housesigma/hr-interview/blob/main/DevOps.md)的回答。
> 由于篇幅原因，这里不再写完整的题目内容，只通过标题来区分。

#### 一、写一个定时执行的Bash脚本，每月的一号凌晨1点 对 MongoDB 中 test.user_logs 表进行备份、清理
backup_mongo_user_logs.sh
```bash
#!/bin/bash

# 这个脚本假设MongoDB 已经配置好，并且服务器上已经安装了必要的工具（如 mongodump、tar 等）。

# 检测执行结果并触发 webhook
function check_status {
  if [ $? -ne 0 ]; then
    curl -X POST https://monitor.ipo.com/webhook/mongodb
    exit 1
  fi
}

# 获取上个月的年份和月份
last_month=$(date -d "last month" +%Y-%m)

# 备份 MongoDB 数据到本地
mongodump --db test --collection user_logs --query "{create_on: {\$lt: new Date('$last_month-01T00:00:00Z')}}" --out /tmp/mongobackup
check_status

# 打包备份文件
tar -zcvf /tmp/mongobackup.tar.gz /tmp/mongobackup
check_status

# 使用 sftp 传输备份文件到远程服务器
sftp bak@bak.ipo.com <<EOF
cd backup_directory
put /tmp/mongobackup.tar.gz
exit
EOF
check_status

# 清理备份数据
mongo --eval "db.user_logs.remove({create_on: {\$lt: new Date('2024-01-01T03:33:11Z')}})" test
check_status

# 清理临时文件
rm -rf /tmp/mongobackup /tmp/mongobackup.tar.gz
```
将如下定时任务写入对应的执行用户的crontab中，例如root的文件路径是：/var/spool/cron/root
```
00 01 01 * * /bin/bash /path/to/backup_mongo_user_logs.sh
```

#### 二、根据要求提供一份Nginx配置
```nginx
server {
    listen 80;
    server_name ipo.com;
    # 非http请求经过301重定向到https
    return 301 https://$host$request_uri;
}

server {
    # 域名：ipo.com, 支持https、HTTP/2
    listen 443 ssl http2;
    server_name ipo.com;

    ssl_certificate /path/to/your/fullchain.pem;
    ssl_certificate_key /path/to/your/privatekey.pem;

    location / {
        # 根据UA进行判断，如果包含关键字 "Google Bot", 反向代理到 server_bot[bot.ipo.com] 去处理
        if ($http_user_agent ~* "Google Bot") {
            proxy_pass http://bot.ipo.com;
            break;
        }
        try_files $uri $uri/ /index.html /public/index.html /api/landing;
    }

    location /api/ {
        # 限流
        limit_req zone=api_limit burst=3 nodelay;
        limit_req_status 429;
        try_files $uri @api;
    }

    location @api {
        # /api/{name} 路径的请求通过unix sock发送到本地 php-fpm，文件映射 /www/api/{name}.php
        include fastcgi_params;
        fastcgi_pass unix:/var/run/php-fpm.sock;
        fastcgi_param SCRIPT_FILENAME /www/api/$name.php;
    }

    location /static/ {
        # 静态文件的一些优化配置，如开启 gzip、添加缓存等
    }

    # 其他配置，如 SSL 配置、日志配置等
}

# 限流配置
limit_req_zone $binary_remote_addr zone=api_limit:10m rate=1.5r/s;
```
