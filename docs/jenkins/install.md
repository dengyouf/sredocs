# 安装

```shell
mkdir -pv   /apps/jenkins && cd /apps/jenkins && mkdir jenkins_home

cat > docker-compose.yml <<'EOF'
version: '3.8'

services:
  jenkins:
    image: jenkins/jenkins:lts
    container_name: jenkins
    privileged: true
    ports:
      - "8080:8080"
      - "50000:50000"
    environment:
      - TZ=Asia/Shanghai
    volumes:
      - ./jenkins_home:/var/jenkins_home
      - /etc/localtime:/etc/localtime:ro  # 保证宿主机与容器时区一致
    restart: always

volumes:
  jenkins_home:
EOF

docker compose up -d
```



