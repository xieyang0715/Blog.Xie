---
title: sonarqube分析代码
date: 2021-03-24 08:41:36
tags:
- sonarqube
---



# java

```bash
# 代码目录
/usr/local/sonar-scanner/bin/sonar-scanner -Dsonar.projectKey=platform_saas -Dsonar.host.url=http://192.168.0.77:9000 -Dsonar.language=java -Dsonar.projectVersion=1.0 -Dsonar.sourceEncoding=UTF-8 -Dsonar.java.binaries=target/classes -Dsonar.sources=src/main/java
```

# js

```bash
# 代码目录
/usr/local/sonar-scanner/bin/sonar-scanner -Dsonar.projectKey=admin-web -Dsonar.host.url=http://192.168.0.77:9000 -Dsonar.language=js -Dsonar.projectVersion=1.0 -Dsonar.sourceEncoding=UTF-8 -Dsonar.sources=./
```

