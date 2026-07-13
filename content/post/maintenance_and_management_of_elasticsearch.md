---
date: 2026-06-14 10:00:00
title: elasticsearch 维护和管理
tags:
  - "elasticsearch"
draft: false
---

<!--more-->

```bash

                                       (@@) (  ) (@)  ( )  @@    ()    @     O     @     O      @
                                  (   )
                              (@@@@)
                           (    )

                         (@@@)
                       ====        ________                ___________
                   _D _|  |_______/        \__I_I_____===__|_________|
                    |(_)---  |   H\________/ |   |        =|___ ___|      _________________
                    /     |  |   H  |  |     |   |         ||_| |_||     _|                \_____A
                   |      |  |   H  |__--------------------| [___] |   =|                        |
                   | ________|___H__/__|_____/[][]~\_______|       |   -|                        |
                   |/ |   |-----------I_____I [][] []  D   |=======|____|________________________|_
                 __/ =| o |=-~O=====O=====O=====O\ ____Y___________|__|__________________________|_
                  |/-=|___|=    ||    ||    ||    |_____/~\___/          |_D__D__D_|  |_D__D__D_|
                   \_/      \__/  \__/  \__/  \__/      \_/               \_/   \_/    \_/   \_/

```

# 设置 transport 证书

适用于单节点集群扩展的情况，重新统一配置集群节点之间的 transport 证书。

一般来说，使用官方软件包安装的单节点集群，有效的配置文件都是如下内容：

```yaml
path.data: /var/lib/elasticsearch
path.logs: /var/log/elasticsearch
xpack.security.enabled: true
xpack.security.enrollment.enabled: true
xpack.security.http.ssl:
  enabled: true
  keystore.path: certs/http.p12
xpack.security.transport.ssl:
  enabled: true
  verification_mode: certificate
  keystore.path: certs/transport.p12
  truststore.path: certs/transport.p12
cluster.initial_master_nodes: ["node"]
http.host: 0.0.0.0
```

安装后默认会自动生成证书，严格来说，所有的 SSL 都应该开启，但是这里只涉及 transport 部分，并且 http SSL 可以根据实际情况来决定是否开启。

执行以下步骤，将单节点的 transport 证书替换，然后重启集群。

```bash
# 所有证书生成步骤的密码都应该妥善保存
# 生成 CA 证书，不做特别设置的情况，生成的证书一般默认在 /usr/share/elasticsearch/ 下
/usr/share/elasticsearch/bin/elasticsearch-certutil ca

# 签发证书，使用前面生成的 CA 证书来进行签发
/usr/share/elasticsearch/bin/elasticsearch-certutil cert --ca elastic-stack-ca.p12

# 修改配置文件中的 transport.ssl 部分的 keystore.path 和 truststore.path ，指向上一步生成的 p12 文件

# 查看原有 p12 证书的密码
/usr/share/elasticsearch/bin/elasticsearch-keystore show xpack.security.transport.ssl.keystore.secure_password
/usr/share/elasticsearch/bin/elasticsearch-keystore show xpack.security.transport.ssl.truststore.secure_password

# 删除原有 p12 证书的密码
/usr/share/elasticsearch/bin/elasticsearch-keystore remove xpack.security.transport.ssl.keystore.secure_password
/usr/share/elasticsearch/bin/elasticsearch-keystore remove xpack.security.transport.ssl.truststore.secure_password

# 添加新的 p12 证书的密码，注意这里是签发证书时的密码，不是 CA 证书的密码
/usr/share/elasticsearch/bin/elasticsearch-keystore add xpack.security.transport.ssl.keystore.secure_password
/usr/share/elasticsearch/bin/elasticsearch-keystore add xpack.security.transport.ssl.truststore.secure_password
```

完成之后，需要在每一台新加入集群的节点机器上使用同一个 CA 证书进行证书签发，修改对应的证书文件权限，然后再修改节点的配置和对应的证书密码，这样就可以顺利加入原有的单节点集群。

一些可以参考的官方文档：

- <https://www.elastic.co/docs/deploy-manage/maintenance/add-and-remove-elasticsearch-nodes>
- <https://www.elastic.co/docs/deploy-manage/security/set-up-basic-security>

# 修改集群索引上限

在集群节点比较少，但是想容纳更多索引的情况下可以使用。

```json
GET _cluster/stats?filter_path=indices.shards.total

PUT _cluster/settings
{
  "persistent" : {
    "cluster.max_shards_per_node": 5000
  }
}
```

# 索引的生命周期管理

设置了生命周期的索引在执行 rollover 之后，索引的时间值 age 会自动更新，导致 ilm 的过期时间没有办法正常对上，这里参考旧版本 Elasticsearch 的方法，将日期时间写定在索引名称上，可以作为生命周期管理的依据。

```json
# 获取现有的索引模板
GET _index_template/template_a
{
  "index_templates": [
    {
      "name": "template_a",
      "index_template": {
        "index_patterns": [
          "applog-*"
        ],
        "template": {
          "settings": {
            "index": {
              "lifecycle": {
                "name": "lifecycle_a",
                "rollover_alias": "applog"
              },
              "mode": "standard",
              "number_of_shards": "3",
              "number_of_replicas": "1"
            }
          }
        },
        "composed_of": [
          "template_1"
        ],
        "ignore_missing_component_templates": [],
        "created_date_millis": 1776068771375,
        "modified_date_millis": 1779267221213
      }
    }
  ]
}

# 更新索引模板，将原有的 settings 部分复制并额外添加 parse_origination_date 的配置
PUT _index_template/template_a
{
  "template": {
    "settings": {
      "index": {
        "lifecycle": {
          "name": "lifecycle_a",
          "rollover_alias": "applog",
          "parse_origination_date": true
        },
        "mode": "standard",
        "number_of_shards": "3",
        "number_of_replicas": "1"
      }
    }
  }
}

# 关闭当前写入索引，这里的索引匹配了原先已有的模板
POST _aliases
{
  "actions": [
    {
      "add": {
        "index": "applog-000001",
        "alias": "applog",
        "is_write_index": false
      }
    }
  ]
}

# 引用新的索引模板，创建带有日期信息的索引
# 以下是 url 编码结果，实际的请求 url 内容是 <applog-{now/d}-000001>
PUT %3Capplog-%7Bnow%2Fd%7D-000001%3E
{
  "aliases": {
    "applog": {
      "is_write_index": true
    }
  }
}
```

要检验生命周期的轮转效果，可以调整生命周期的检查频率，默认的检查频率是 `10m` ，验证完成后应该调整回来，这个检查频率不应该过高。

```json
PUT _cluster/settings
{
  "persistent": {
    "indices.lifecycle.poll_interval": "1m"
  }
}
```

# 自定义日期格式的组件模板

通过使用自定义日期格式的组件模板，应用于索引模板中。

```json
# 获取当前组件模板配置
GET /_component_template/template_1

# 修改当前组件模板配置
# 使用日期格式 yyyy-MM-dd HH:mm:ss.SSSXXX 对应的具体时间值 2026-06-21 15:32:25.613+08:00
PUT /_component_template/template_1
{
  "template": {
    "mappings": {
      "properties": {
        "log": {
          "type": "text"
        },
        "date": {
          "type": "date",
          "format": "yyyy-MM-dd HH:mm:ss.SSSXXX||strict_date_optional_time||epoch_millis"
        }
      }
    }
  }
}
```

在不使用模板的情况下，可以直接通过创建索引进行测试。

```json
PUT my-index-000001
{
  "mappings": {
    "properties": {
      "log": {
        "type": "text"
      },
      "date": {
        "type": "date",
        "format": "yyyy-MM-dd HH:mm:ss.SSSXXX||strict_date_optional_time||epoch_millis"
      }
    }
  }
}

POST /my-index-000001/_doc/
{
  "log": "log",
  "date": "2026-06-21 15:32:25.613+08:00"
}
```
