## grafana安装
* [下载地址](https://grafana.com/grafana/download)
* 安装根据想要的操作系统只要解压压缩包即可
* 配置数据源

  <img src="https://github.com/ZXBmmt/blog/blob/master/resources/spring-actuator/grafana_config_datasource.png" width="35%" height="35%"></img>
* 配置开启邮件报警
```
##编辑 {data}/grafana/conf
#################################### SMTP / Emailing #####################
[smtp]
enabled = true
host = smtp.139.com:25
user = xxx@139.com
# If the password contains # or ; you have to wrap it with triple quotes. Ex """#password;"""
password = xxxx
cert_file =
key_file =
skip_verify = true
from_address = xxx@139.com
from_name = Grafana
ehlo_identity =
[alerting]
# Makes it possible to turn off alert rule execution.
execute_alerts = true
```
* 配置发送邮件邮箱

  <img src="https://github.com/ZXBmmt/blog/blob/master/resources/spring-actuator/config_email.png" width="80%" height="80%"></img>

## 为grafana安装数据源graphite
* 安装教程见:安装过程请参考[陌生人——蓝精灵:《graphite安装（一键搞定版）》](https://blog.csdn.net/liuxiao723846/article/details/82735147)

## 使用spring-actuator
* [源代码](https://github.com/ZXBmmt/spring-actuator)
* spring-boot-starter-actuator依赖

```java
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```
* 2.application.properties配置

```properties
# management 监控端点http访问根路径
management.endpoints.web.base-path=/monitor
#允许暴露的端点
management.endpoints.web.exposure.include=info,health,metrics,env
management.endpoint.health.show-details=always
## metrics信息保留时长
management.endpoint.metrics.cache.time-to-live=604800s
```
* 以metrics为例

```java
@GetMapping("/helloWorld")
public String helloWorld(){
    String humanMinute = sdf.format(new Date());
    Metrics.counter("test", "day", humanMinute).increment(1);
    return "hello world";
}
```
## 监控数据拉取

* 使用python脚本访问监控集群各节点的指标数据，统计拉取到graphite，脚本定时执行

```python
import datetime
import json
import socket
import time
import urllib.request

CARBON_SERVER = '10.1.7.45'  # grafana/graphite/carbon server
CARBON_PORT = 2003
carbon_db = ''

serverList = ['http://127.0.0.1:8080']

#function: read today's collected count
def readMetricTodayCount(serverMetricsUrl1,todayHuman1):
    fullMetricsUrl = serverMetricsUrl1 + "?tag=day:" + todayHuman1
    try:
        collectedCount = json.loads(urllib.request.urlopen(fullMetricsUrl, None, 30).read())
        logNum = collectedCount['measurements'][0]['value']
        return logNum
    except:
        return 0

#main function
sock = socket.socket()
sock.connect((CARBON_SERVER, CARBON_PORT))
metrics_ary = []

for server in serverList:
    serverMetricsUrl = ''
    #format to http-127-0-0-1-8080
    graphiteServerMetricsUrl = server.replace('://', '-').replace(':', '-')
    if server.endswith('/'):
        serverMetricsUrl = server + 'monitor/metrics'
        graphiteServerMetricsUrl = graphiteServerMetricsUrl.replace('.', '-').replace('/', '')
    else:
        serverMetricsUrl = server + '/monitor/metrics'
        graphiteServerMetricsUrl = graphiteServerMetricsUrl.replace('.', '-')
    todayHuman = datetime.datetime.now(datetime.timezone(datetime.timedelta(hours=8))).strftime('%Y-%m-%d')
    hellWorldCallCountMetricsReqUrl = serverMetricsUrl + "/test"
    hellWorldCallSuccessCount = readMetricTodayCount(hellWorldCallCountMetricsReqUrl,todayHuman)
    print(hellWorldCallCountMetricsReqUrl)
    print(hellWorldCallSuccessCount)
    graphiteKeyPullOpmTaskRunSuccessCountToday = carbon_db + "helloworld.success.count." + graphiteServerMetricsUrl + ".today"
    graphiteKeyPullOpmTaskRunSuccessCountDay = carbon_db + "helloworld.success.count." + graphiteServerMetricsUrl + "." + todayHuman
    ts = int(round(time.time()))
    metrics_ary.append('%s %s %d' % (graphiteKeyPullOpmTaskRunSuccessCountToday, hellWorldCallSuccessCount, ts))
    metrics_ary.append('%s %s %d' % (graphiteKeyPullOpmTaskRunSuccessCountDay, hellWorldCallSuccessCount, ts))
msg = '\n'.join(metrics_ary) + '\n'
#print(msg)
sock.sendall(msg.encode())
sock.close()
```
* 简单测试
```
xbZhangdeMacBook-Pro:scripts zxb$ curl http://127.0.0.1:8080/helloWorld
xbZhangdeMacBook-Pro:scripts zxb$ python3 metric-to-carbon.py
```
<img src="https://github.com/ZXBmmt/blog/blob/master/resources/spring-actuator/test_collect_metrics.png" width="80%" height="80%"></img>

## 监控脚本定时执行
* linux操作系统
```
crontab -l //查看定时任务
crontab -e //编辑定时任务列表
#每分钟执行
*/1 * * * * /usr/local/bin/python3 /data/spring-actuator/scripts/metrics-to-carbon.py >> /data/spring-actuator/sys.log
```

## grafana展示数据设置报警

* 配置查询面板
  <img src="https://github.com/ZXBmmt/blog/blob/master/resources/spring-actuator/config_grafana_panel.png" width="80%" height="80%"></img>

  <img src="https://github.com/ZXBmmt/blog/blob/master/resources/spring-actuator/config_alter.png" width="80%" height="80%"></img>
* 报警效果
  小贴士:139邮箱可以开启短息通知
    <img src="https://github.com/ZXBmmt/blog/blob/master/resources/spring-actuator/config_alter.png" width="80%" height="80%"></img>
