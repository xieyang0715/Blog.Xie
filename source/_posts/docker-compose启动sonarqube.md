---
title: "docker-compose启动sonarqube"
date: 2021-03-22 00:43:28
tags:
- "sonarqube"
- "docker-compose"
---

访问url: `http://<your ip>:9000`

用户名: sonarqube, 密码：123456

```bash
# .env
PGP=123456
PGU=sonarqube
```

```yaml
[root@izbp1iegoe89wvmbuy7ky1z sonarqube]# cat docker-compose.yaml 
version: '3'
services:
  postgresql:
    image: postgres:12.3-alpine
    volumes:
      - ./postgresql_data:/var/lib/postgresql/data
      - ./postgresql:/var/lib/postgresql/
    restart: always
    environment:
      - POSTGRES_PASSWORD=${PGP}
      - POSTGRES_USER=${PGU}
      - POSTGRES_DB=sonar
    networks:
      - sonarqubenet
  sonarqube:
    image: sonarqube:8-community
    depends_on:
    - postgresql
    container_name: sonar
    restart: always
    ports:
      - 8599:9000
    volumes:
      - ./sonarqube_extensions:/opt/sonarqube/extensions
      - ./sonarqube_data:/opt/sonarqube/data
      - ./sonarqube_logs:/opt/sonarqube/logs
    privileged: true
    environment:
      - SONARQUBE_JDBC_URL=jdbc:postgresql://postgresql:5432/sonar
      - SONARQUBE_JDBC_PASSWORD=${PGP}
      - SONARQUBE_JDBC_USERNAME=${PGU}
    networks:
      - sonarqubenet
networks:
  sonarqubenet:
```



安装中文插件