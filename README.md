## 防护配置
- 将 action 配置成 cmd 命令，执行防护操作
- 或将 action 配置成 webhook 接口，发送防护操作

### 四层防护
- 先决条件： 启动 syn_collector 来抓流量
```
./syn_collector -h
Usage: syn_collector [OPTIONS]

Options:
      --iface <IFACE>        [default: eth0]
      --socket <SOCKET>      [default: /tmp/syn.sock]
      --flush-ms <FLUSH_MS>  [default: 1000]
  -h, --help                 Print help
```
- --iface 网络接口，默认 eth0
- --socket 与 fluent-bit 通信的 socket 文件，默认 /tmp/syn.sock
- --flush-ms 缓存多久后，将缓存中的数据刷新到 fluent-bit，默认 1000ms

#### 场景1： 当某个客户端2秒内发起1000次连接时（syn+udp+icmp），触发防护
- 启动网卡的流量监控
```
./syn_collector --iface eth0 --socket /tmp/syn.sock --flush-ms 1000
```
- 配置统计规则文件 rules.json
```json
[{
    "id": "xxx-yyy-zzz",
    "reset_cycle": "*/2 * * * * *",
    "name": "2秒1000次连接",
    "is_enabled":true,
    "message": "2秒内客户端 发起了1000次连接(一个syn包算一次，一个udp或icmp包算一次)，则发送信息到 udp 127.0.0.1:1234 端口",
    "actions": [
      ["udp","127.0.0.1:1234"]
    ],
    "aggregates": [
      ["count", ">", 1000, {"$mode":"sum", "$group":["src"]}]
    ]
}]
```
- 配置 fluent-bit 规则文件
```yaml
plugins:
  - ./fluent-bit-cc.so

service:
    parsers_file: /etc/fluent-bit/parsers.conf

pipeline:
  inputs:
    - name: syslog
      path: /tmp/syn.sock
      mode: unix_udp
      parser: syslog-rfc3164-local
      unix_perm: 0666

  filters:
    - name: parser
      match: "*"
      key_name: "message"
      parser: json

  outputs:
    - name: fluent-bit-stats
      match: "*"
      config: "./examples/rules.json"    # 配置文件
      interval: 5s                       # 检查配置文件的改动间隔时间，当文件改动时，自动重新加载
      debug: trace                       # 是否开启调试模式，默认关闭，可选值有： trace, debug, info, warn, error
```
- 启动 fluent-bit
```
fluent-bit ./example/fluent-bit.yaml
```

#### 场景2： 当某佧客户端10秒内发送的数据超过100MB时，触发防护
- 修改配置，只需将 rules.json 中的 aggregates 字段修改为下面即可
- 相当于 where sum(bytes) > 100MB group by src
```json
 "aggregates": [
      ["bytes", ">", 1000000000, {"$mode":"sum", "$group":["src"]}]
    ]
```

#### 场景3： 只对某些IP段进行防护
- 修改配置，只需将 rules.json 中的 conditions 字段修改为下面即可
- 默认的 conditions 是 and 关系，若需要 or 关系，需在 conditions 字段前添加 "or"
```json
 "conditions": [ "or",
      ["src", "incidr", "1.1.1.0/24"],
      ["src", "incidr", "2.2.2.0/24"]
    ]
```


### 七层防护

#### 场景1： 对 各种http服务 的访问日志进行统计防护
- 支持 apache, apache2, nginx, apache_error, k8s-nginx-ingress
- 配置fluent-bit的input输入 + fluent-bit的filter转成json，再输出到 fluent-bit-stats 插件进行统计

#### 场景2： 单个IP访问、单元时间内的请求次数
- 类sql: where count(client_ip) > 100 group by client_ip
- 统计 client_ip 字段的访问次数
```json
    "aggregates": [
      ["client_ip", ">", 100, {"$mode":"count", "$group":["client_ip"]}]
    ]
```

#### 场景3： 某个网站、单元时间内的请求次数
- 类sql: where count(host) > 100 group by host, client_ip
- 统计 host 字段的访问次数
```json
    "aggregates": [
      ["host", ">", 100, {"$mode":"count", "$group":["host", "client_ip"]}]
    ]
```

#### 场景4： 某个来源请求数太多
- 统计 referer 字段的访问次数
```json
    "aggregates": [
      ["referer", ">", 100, {"$mode":"count", "$group":["referer"]}]
    ]
```

#### 场景5： 第888个签到的人
- 统计 client_ip 字段的访问次数
```json
    "aggregates": [
      ["client_ip", "=", 888, {"$mode":"count", "$group":["client_ip"]}]
    ],
    "conditions": [
      ["host", "=", "www.a.com"],
      ["request_path", "=", "/sign"]
    ]
```

#### 场景6： 每天开放10000个国久IP访问（需要再编译nginx，开启 maxmind 支持）
- 统计 client_ip 字段的访问次数
- https://nginx.org/en/docs/http/ngx_http_geoip_module.html#geoip_city
```json
    "aggregates": [
      ["client_ip", ">", 10000, {"$mode":"count", "$group":["client_ip"]}]
    ],
    "conditions": [
      ["geoip_city_country_code", "=", "US"]
    ]
```
