## 安装手册
### docker 部署 jira

```shell
hostnamectl set-hostname --static jira
echo "192.168.0.117 jira" >> /etc/hosts
```

```shell
docker pull atlassian/jira-software:8.16.1
docker pull mysql:8.0.22
```

```shell
docker run -e "MYSQL_ROOT_PASSWORD=123123" -d --name mysql -p 3306:3306 -v ./mysql/data:/var/lib/mysql mysql:8.0.22
docker exec -it mysql mysql -uroot -p123456 -e "CREATE DATABASE jiradb CHARACTER SET utf8mb4 COLLATE utf8mb4_bin;"
docker exec -it mysql mysql -uroot -p123456 -e "SHOW DATABASES;"
mysql: [Warning] Using a password on the command line interface can be insecure.
+--------------------+
| Database           |
+--------------------+
| information_schema |
| jiradb             |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
```
```shell
mkdir -pv  /data/apps/jira
docker run -d --name jira -p 8080:8080 -v /data/apps/jira/data:/var/atlassian/application-data/jira atlassian/jira-software:8.16.1
```