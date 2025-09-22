---
date: 2024-02-18 15:00:00
title: 通过 cloudflare api 更新 DNS 记录
tags:
  - "cloudflare"
  - "dns"
  - "python"
draft: false
---

通过 Cloudflare API 更新 DNS 记录的 python 脚本。

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

```python
import urllib.parse
import urllib.request
import json

headers = {
    'Authorization': 'Bearer TOKEN_CONTENT',
    'Content-Type': 'application/json',
}

def get_zone_id(domain):
    url = 'https://api.cloudflare.com/client/v4/zones'
    req = urllib.request.Request(url, data=None, headers=headers)
    with urllib.request.urlopen(req) as response:
        body = json.loads(response.read().decode('utf-8'))
        for i in body.get('result'):
            if i.get('name') == domain:
                return i.get('id')


def get_dns_record_id(domain_id, record):
    url = f'https://api.cloudflare.com/client/v4/zones/{domain_id}/dns_records'
    req = urllib.request.Request(url, data=None, headers=headers)
    with urllib.request.urlopen(req) as response:
        body = json.loads(response.read().decode('utf-8'))
        for i in body.get('result'):
            if i.get('name') == record:
                return i.get('id')


def put_dns_record_A(domain_id, record_id, name, content):
    url = f'https://api.cloudflare.com/client/v4/zones/{domain_id}/dns_records/{record_id}'
    data = {
        'type': 'A',
        'name': name,
        'content': content,
        'ttl': 60,
        'proxied': False,
    }
    req = urllib.request.Request(url, data=json.dumps(data).encode(), headers=headers, method='PUT')
    with urllib.request.urlopen(req) as response:
        body = json.loads(response.read().decode('utf-8'))
        return body

# def get_ip_addr()
#     req = urllib.request.Request('http://4.ipw.cn')
#     with urllib.request.urlopen(req) as response:
#         body = response.read().decode('utf-8')
#         return body

zone = 'yourdomain.com'
record_name = 'sub.yourdomain.com'
# new_addr = get_ip_addr()
new_addr = '1.1.1.1'
zone_id = get_zone_id(zone)
dns_record_id = get_dns_record_id(zone_id, record_name)
put_dns_record_A(zone_id, dns_record_id, record_name, new_addr)
```
