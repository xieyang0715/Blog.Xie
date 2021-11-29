---
title: sonar判断
date: 2021-10-11 16:17:09
tags:
- sonarqube
---



# 前言

sonarqube在扫描完之后，需要判断扫描状态进行下一步操作

https://docs.sonarqube.org/latest/extend/web-api/

https://docs.sonarqube.org/6.7/MetricDefinitions.html
https://docs.sonarqube.org/latest/user-guide/metric-definitions/

```bash
https://docs.sonarqube.org/6.7/MetricDefinitions.html
https://docs.sonarqube.org/latest/user-guide/metric-definitions/



echo jenkins获取bug
curl -u admin:user_token -s 'http://sonarqube_host:sonarqube_port/api/issues/search?projectKeys=test-pipeline1&types=BUG' | jq -r .total

 curl -u admin:user_token -s 'http://sonarqube_host:sonarqube_port/api/measures/component?component=test-pipeline1&metricKeys=bugs'  | jq -r .component.measures[].value

 curl -u admin:user_token -s 'http://sonarqube_host:sonarqube_port/api/measures/component?component=test-pipeline1&metricKeys=new_bugs'  | jq -r .component.measures[].value

echo 漏洞
curl -u admin:user_token -s 'http://sonarqube_host:sonarqube_port/api/issues/search?projectKeys=test-pipeline1&types=VULNERABILITY' | jq -r .total


echo code smell
curl -u admin:user_token -s 'http://sonarqube_host:sonarqube_port/api/issues/search?projectKeys=test-pipeline1&types=CODE_SMELL' | jq -r .total





echo 获取代码重复代码率
curl -u admin:user_token -s 'http://sonarqube_host:sonarqube_port/api/measures/component?component=test-pipeline1&metricKeys=duplicated_lines_density' | jq -r .component.measures[].value


echo 代码总行数
curl -u admin:user_token -s 'http://sonarqube_host:sonarqube_port/api/measures/component?component=test-pipeline1&metricKeys=ncloc'  | jq -r .component.measures[].value


curl -u admin:user_token -s 'http://sonarqube_host:sonarqube_port/api/measures/component?component=test-pipeline1&metricKeys=complexity'
curl -u admin:user_token -s 'http://sonarqube_host:sonarqube_port/api/measures/component?component=test-pipeline1&metricKeys=violations'




代码安全：
<!--more-->
# 新代码的代码深层问题
curl -u admin:user_token -s 'http://sonarqube_host:sonarqube_port/api/measures/component?component=test-pipeline1&metricKeys=new_code_smells'  | jq -r .component.measures[].period.value
网址
http://sonarqube_host:sonarqube_port/project/issues?id=test-pipeline1&resolved=false&sinceLeakPeriod=true&types=CODE_SMELL


# 新代码的漏洞
curl -u admin:user_token -s 'http://sonarqube_host:sonarqube_port/api/measures/component?component=test-pipeline1&metricKeys=new_vulnerabilities'  | jq -r .component.measures[].period.value
网址
http://sonarqube_host:sonarqube_port/project/issues?id=test-pipeline1&resolved=false&sinceLeakPeriod=true&types=VULNERABILITY


可靠性
# 新代码的BUG
curl -u admin:user_token -s 'http://sonarqube_host:sonarqube_port/api/measures/component?component=test-pipeline1&metricKeys=new_bugs'  | jq -r .component.measures[].period.value
网址
http://sonarqube_host:sonarqube_port/project/issues?id=test-pipeline1&resolved=false&sinceLeakPeriod=true&types=BUG

质量状态
curl -u admin:user_token -s 'http://sonarqube_host:sonarqube_port/api/measures/component?component=test-pipeline1&metricKeys=alert_status'  | jq -r .component.measures[].value
网址
http://sonarqube_host:sonarqube_port/dashboard?id=test-pipeline1

质量未通过
curl -u admin:user_token -s 'http://sonarqube_host:sonarqube_port/api/measures/component?component=test-pipeline1&metricKeys=alert_status'  | jq -r .component.measures[].value
```

# jenkins中引用

sonarqube执行结果 

> ```xml
> INFO: Analysis report uploaded in 364ms
> INFO: ANALYSIS SUCCESSFUL, you can browse http://sonarqube_host:sonarqube_port/dashboard?id=cloudweb-ci
> INFO: Note that you will be able to access the updated dashboard once the server has processed the submitted analysis report
> INFO: More about the report processing at http://sonarqube_host:sonarqube_port/api/ce/task?id=AXxuX08jnh20tsgQhBBf
> ```

获取jenkins中输出的sonarqube执行结果的id, 并检查id对应的sonarqube上的状态。

- 成功就继续，失败就退出，其它就循环

```bash
echo 获取扫描结果 
# id来自jenkins: 
# wget -qO- --content-on-error --no-proxy --auth-no-challenge --http-user=**** --http-password=**** https://****.baidu.com/job/cloudweb-ci/794/consoleText | grep 'More about the report processing' | head -n1
# 返回 AXxuX08jnh20tsgQhBBf


# wget -qO- --content-on-error --no-proxy --auth-no-challenge --http-user=**** --http-password= http://sonarqube_host:sonarqube_port/api/ce/task?id=AXxuX08jnh20tsgQhBBf | jq -r .status | jq -r .task
# OK/ERROR/FAILED/IN_PROGRESS
```

```bash
while true; do
	wget -qO- --content-on-error --no-proxy --auth-no-challenge --http-user=**** --http-password=**** https://****.baidu.com/job/cloudweb-ci/794/consoleText | grep 'More about the report processing' | head -n1
	case status:
	ok)
	error)
	in_progress)
		continue
	failed)
do
```

获取sonarqube质量状态

- 新项目成功继续，失败就退出

```bash
echo 代码质量状态
curl -u admin:user_token -s 'http://sonarqube_host:sonarqube_port/api/measures/component?component=test-pipeline1&metricKeys=alert_status' | jq -r .component.measures[].value
// OK/ERROR
```

