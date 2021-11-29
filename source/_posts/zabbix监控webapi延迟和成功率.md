---
title: zabbix监控webapi延迟和成功率
date: 2021-09-10 09:46:33
tags:
- zabbix
- web监测
- 生产小经验
---



# 监测状态

![image-20210910094716224](http://myapp.img.mykernel.cn/image-20210910094716224.png)



报警配置

```bash
web的状态延迟大于800ms已恢复!
告警主机:重构正式服务器A
告警地址:x.x.x.x
监控项目:web的状态 延迟
监控取值:18.25 ms
告警等级:Average
当前状态:OK
告警时间:2021.09.10 06:34:18
恢复时间:2021.09.10 06:35:49
持续时间:1m
事件ID:49391547 @所有人 
```

今天9月15日，安全工程师删除了集群通信地址的安全组，收到此监控的报警。我们快速定位后端问题，redis问题均正常，就怀疑安全组的问题，查看操作记录，原来是集群安全组被删除了

![image-20210915110313230](http://myapp.img.mykernel.cn/image-20210915110313230.png)

# 实现方法

获取nginx日志中的request_time字段，获取1m平均延迟。

获取nginx日志中的status字段，获取1m，非5xx占总比率。

```bash
# 获取1m的nginx所有日志，此为阿里云ACK的k8s ingress日志获取方法, 其他ingress 方法一致。
kubectl logs --since=1m deploy/nginx-ingress-controller --all-containers=true --kubeconfig=/etc/zabbix/scripts/readlogconfig  -n kube-system  2>/dev/null  |  awk -F'"' '/^[0-9]+/ {print}')

# 过滤request_time字段
求1m所有此字段的和，除以请求数，就是单次请求的延迟


# 过滤status字段
求非5xx所有个数，除以总请求数，再乘100，就是成功率
```

> 返回的延迟为ms, 成功率为%



# 配置

1. zabbix配置

   ```bash
   UserParameter=api.stat[*],/etc/zabbix/scripts/api.stat "$1"
   ```

2. 准备`api.stat`脚本

   ```bash
   #!/bin/bash
   # script.sh lantency /api/Verson/getversonbyproductid
   # script.sh success_rate /api/Verson/getversonbyproductid
   export OP=$1
   export URL=$2
   
   
   content="$(kubectl logs --since=1m deploy/nginx-ingress-controller --all-containers=true --kubeconfig=/etc/zabbix/scripts/readlogconfig  -n kube-system  2>/dev/null  |  awk -F'"' '/^[0-9]+/ {print}')"
   
   if [ "$OP" == "lantency" ]; then
           # 最近1分钟的平均延迟 ms
           newcontent="$(echo "$content" | awk -F'"' '/^[0-9]+/{print $7}')" 
           if [ -z "$newcontent" ]; then
                   echo 0
                   exit
           fi
           echo "$newcontent" | awk '{count++;sum+=$2}END{if (count=="0"){print 0} else { res=sum/count; if (res~"nan"){print 0} else {printf "%.2f\n",sum/count*1000}}}'
   elif [ "$OP" == "success_rate" ]; then
           # 最近1分钟的成功率 %
           newcontent="$(echo "$content" | awk -F'"' '/^[0-9]+/{print $3}')" 
           echo "$newcontent" | awk '{if ($1 !~ "5.*") {scount++}; count++}END{res=scount/count*100; if (res ~ "nan"){print 100} else {print res}}'
   fi
   ```

3. 获取k8s的认证文件

   要求权限对kube-system名称空间的nginx-ingress只读

   - 获取token

     创建用户，绑定kube-system名称空间的view, 获取token

   ```bash
   kubectl create sa -n kube-system readlog
   kubectl create -n kube-system rolebinding readlog --clusterrole=view  --serviceaccount=kube-system:readlog
   kubectl describe  -n kube-system  secret readlog-token-xxxx
   ```

   - 配置token给kubeconfig

     ```bash
     cp /etc/kubernetes/admin.config ./readlogconfig
     ```

     清理用户下面的的认证

     ```yaml
     users:
     - name: "kube"
       user:
     ```

     添加token

     ```bash
     kubectl config set-credentials kube --token=xxx --kubeconfig=./readlogconfig 
     ```

   - 验证

     ```bash
     pos-prod@1227f652bef0:~$ kubectl get pod --kubeconfig=./readlogconfig 
     Error from server (Forbidden): pods is forbidden: User "system:serviceaccount:kube-system:readlog" cannot list resource "pods" in API group "" in the namespace "default"
     pos-prod@1227f652bef0:~$ kubectl get pod -n kube-system --kubeconfig=./readlogconfig 
     ```

     可以看出对default没有权限，但对kube-system有权限

4. zabbix模板

   `zabbix_4.0_template.xml` 仅对zabbix 4.0 生效，其他版本，自行参考此文件，生成

   导入方法：配置、模板、导入

   ```xml
   <?xml version="1.0" encoding="UTF-8"?>
   <zabbix_export>
       <version>4.0</version>
       <date>2021-09-09T08:06:25Z</date>
       <groups>
           <group>
               <name>kubernetes</name>
           </group>
       </groups>
       <templates>
           <template>
               <template>monitor_nginxingress_api</template>
               <name>监控nginx ingress日志的api</name>
               <description>监控web应用的每一个接口</description>
               <groups>
                   <group>
                       <name>kubernetes</name>
                   </group>
               </groups>
               <applications/>
               <items>
                   <item>
                       <name>web的状态 延迟</name>
                       <type>0</type>
                       <snmp_community/>
                       <snmp_oid/>
                       <key>api.stat[lantency]</key>
                       <delay>30s</delay>
                       <history>90d</history>
                       <trends>365d</trends>
                       <status>0</status>
                       <value_type>0</value_type>
                       <allowed_hosts/>
                       <units>ms</units>
                       <snmpv3_contextname/>
                       <snmpv3_securityname/>
                       <snmpv3_securitylevel>0</snmpv3_securitylevel>
                       <snmpv3_authprotocol>0</snmpv3_authprotocol>
                       <snmpv3_authpassphrase/>
                       <snmpv3_privprotocol>0</snmpv3_privprotocol>
                       <snmpv3_privpassphrase/>
                       <params/>
                       <ipmi_sensor/>
                       <authtype>0</authtype>
                       <username/>
                       <password/>
                       <publickey/>
                       <privatekey/>
                       <port/>
                       <description/>
                       <inventory_link>0</inventory_link>
                       <applications/>
                       <valuemap/>
                       <logtimefmt/>
                       <preprocessing/>
                       <jmx_endpoint/>
                       <timeout>3s</timeout>
                       <url/>
                       <query_fields/>
                       <posts/>
                       <status_codes>200</status_codes>
                       <follow_redirects>1</follow_redirects>
                       <post_type>0</post_type>
                       <http_proxy/>
                       <headers/>
                       <retrieve_mode>0</retrieve_mode>
                       <request_method>0</request_method>
                       <output_format>0</output_format>
                       <allow_traps>0</allow_traps>
                       <ssl_cert_file/>
                       <ssl_key_file/>
                       <ssl_key_password/>
                       <verify_peer>0</verify_peer>
                       <verify_host>0</verify_host>
                       <master_item/>
                   </item>
                   <item>
                       <name>web的状态 成功率</name>
                       <type>0</type>
                       <snmp_community/>
                       <snmp_oid/>
                       <key>api.stat[success_rate]</key>
                       <delay>30s</delay>
                       <history>90d</history>
                       <trends>365d</trends>
                       <status>0</status>
                       <value_type>3</value_type>
                       <allowed_hosts/>
                       <units>%</units>
                       <snmpv3_contextname/>
                       <snmpv3_securityname/>
                       <snmpv3_securitylevel>0</snmpv3_securitylevel>
                       <snmpv3_authprotocol>0</snmpv3_authprotocol>
                       <snmpv3_authpassphrase/>
                       <snmpv3_privprotocol>0</snmpv3_privprotocol>
                       <snmpv3_privpassphrase/>
                       <params/>
                       <ipmi_sensor/>
                       <authtype>0</authtype>
                       <username/>
                       <password/>
                       <publickey/>
                       <privatekey/>
                       <port/>
                       <description/>
                       <inventory_link>0</inventory_link>
                       <applications/>
                       <valuemap/>
                       <logtimefmt/>
                       <preprocessing/>
                       <jmx_endpoint/>
                       <timeout>3s</timeout>
                       <url/>
                       <query_fields/>
                       <posts/>
                       <status_codes>200</status_codes>
                       <follow_redirects>1</follow_redirects>
                       <post_type>0</post_type>
                       <http_proxy/>
                       <headers/>
                       <retrieve_mode>0</retrieve_mode>
                       <request_method>0</request_method>
                       <output_format>0</output_format>
                       <allow_traps>0</allow_traps>
                       <ssl_cert_file/>
                       <ssl_key_file/>
                       <ssl_key_password/>
                       <verify_peer>0</verify_peer>
                       <verify_host>0</verify_host>
                       <master_item/>
                   </item>
               </items>
               <discovery_rules/>
               <httptests/>
               <macros/>
               <templates/>
               <screens/>
           </template>
       </templates>
       <triggers>
           <trigger>
               <expression>{monitor_nginxingress_api:api.stat[lantency].avg(1m)}&gt;=800</expression>
               <recovery_mode>0</recovery_mode>
               <recovery_expression/>
               <name>web的状态延迟大于800ms</name>
               <correlation_mode>0</correlation_mode>
               <correlation_tag/>
               <url/>
               <status>0</status>
               <priority>3</priority>
               <description/>
               <type>0</type>
               <manual_close>0</manual_close>
               <dependencies/>
               <tags/>
           </trigger>
           <trigger>
               <expression>{monitor_nginxingress_api:api.stat[success_rate].avg(1m)}&lt;=99</expression>
               <recovery_mode>0</recovery_mode>
               <recovery_expression/>
               <name>web的状态成功率小于99</name>
               <correlation_mode>0</correlation_mode>
               <correlation_tag/>
               <url/>
               <status>0</status>
               <priority>3</priority>
               <description/>
               <type>0</type>
               <manual_close>0</manual_close>
               <dependencies/>
               <tags/>
           </trigger>
       </triggers>
       <graphs>
           <graph>
               <name>web的状态延迟</name>
               <width>900</width>
               <height>200</height>
               <yaxismin>0.0000</yaxismin>
               <yaxismax>100.0000</yaxismax>
               <show_work_period>1</show_work_period>
               <show_triggers>1</show_triggers>
               <type>0</type>
               <show_legend>1</show_legend>
               <show_3d>0</show_3d>
               <percent_left>0.0000</percent_left>
               <percent_right>0.0000</percent_right>
               <ymin_type_1>0</ymin_type_1>
               <ymax_type_1>0</ymax_type_1>
               <ymin_item_1>0</ymin_item_1>
               <ymax_item_1>0</ymax_item_1>
               <graph_items>
                   <graph_item>
                       <sortorder>0</sortorder>
                       <drawtype>0</drawtype>
                       <color>1A7C11</color>
                       <yaxisside>0</yaxisside>
                       <calc_fnc>2</calc_fnc>
                       <type>0</type>
                       <item>
                           <host>monitor_nginxingress_api</host>
                           <key>api.stat[lantency]</key>
                       </item>
                   </graph_item>
               </graph_items>
           </graph>
           <graph>
               <name>web的状态成功率</name>
               <width>900</width>
               <height>200</height>
               <yaxismin>0.0000</yaxismin>
               <yaxismax>100.0000</yaxismax>
               <show_work_period>1</show_work_period>
               <show_triggers>1</show_triggers>
               <type>0</type>
               <show_legend>1</show_legend>
               <show_3d>0</show_3d>
               <percent_left>0.0000</percent_left>
               <percent_right>0.0000</percent_right>
               <ymin_type_1>0</ymin_type_1>
               <ymax_type_1>0</ymax_type_1>
               <ymin_item_1>0</ymin_item_1>
               <ymax_item_1>0</ymax_item_1>
               <graph_items>
                   <graph_item>
                       <sortorder>0</sortorder>
                       <drawtype>0</drawtype>
                       <color>1A7C11</color>
                       <yaxisside>0</yaxisside>
                       <calc_fnc>2</calc_fnc>
                       <type>0</type>
                       <item>
                           <host>monitor_nginxingress_api</host>
                           <key>api.stat[success_rate]</key>
                       </item>
                   </graph_item>
               </graph_items>
           </graph>
       </graphs>
   </zabbix_export>
   
   ```

   

   
