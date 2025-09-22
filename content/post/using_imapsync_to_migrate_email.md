---
date: 2025-07-07 10:02:45
title: 使用 imapsync 迁移电子邮件
tags:
  - "imap"
draft: false
---

使用 imapsync 迁移电子邮件。

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

通过 imap 协议进行邮件迁移是最通用的邮件迁移方案，一般可以通过 imapsync 这个开源工具来实现。

这里以单个用户的迁移作为例子。

```bash
# 启用一个常驻的 imapsync 容器
docker pull gilleslamiral/imapsync
docker run -dit --name imapsync_instance gilleslamiral/imapsync bash

# 将密码保存到文件并复制到容器中
vi /tmp/password
docker cp /tmp/password imapsync_instance:/var/tmp/password

# 在容器中执行迁移过程
docker exec -it imapsync_instance bash
imapsync --tmpdir /tmp \
  --addheader --noreleasecheck \
  --automap --nofoldersizes --subscribeall \
  --ssl1 --host1 imap.host1.com --port1 993 --user1 user@host1.com --passfile1 /var/tmp/password \
  --ssl2 --host2 imap.host2.com --port2 993 --user2 user@host2.com --passfile2 /var/tmp/password
# --ssl1, --host1, --user1, --passfile1 等带有 1 的选项表示迁移源邮箱
# --ssl2, --host2, --user2, --passfile2 等带有 2 的选项表示迁移目标邮箱
# --addheader 如果邮件不存在 header 信息，则给邮件添加 "Message-Id" header
# --noreleasecheck 不检查 imapsync 更新
# --automap 自动进行文件夹映射
# --nofoldersizes 开始同步前，不对现有的文件目录进行大小计算
# --subscribeall 订阅目标邮箱的所有文件夹，即使它不在源邮箱中
```

有一些邮件服务商已经禁用了用户密码认证，强制采用 OAuth2 认证，比如 office365 ，那么在迁移前需要先获取对应的 token 。

```bash
# 在容器中获取对应账户的 token
docker exec -it imapsync_instance oauth2_imap --provider office365 --remotebrowser user@host2.com
# 此时会给出一个修改过重定向目标的登陆链接，复制到浏览器并输入对应的账户密码后，得到新的链接
# 将新链接中的 code 参数复制出来，输入到终端中，对应的 token 将会以文件形式保存

# 使用 token 执行迁移过程
imapsync --tmpdir /tmp \
  --addheader --noreleasecheck \
  --automap --nofoldersizes --subscribeall \
  --ssl1 --host1 imap.host1.com --port1 993 --user1 user@host1.com --passfile1 /var/tmp/password \
  --ssl2 --host2 outlook.office365.com --port2 993 --user2 user@host2.com \
  --office2 --oauthaccesstoken2 /var/tmp/tokens/oauth2_tokens_user@host2.com.txt 
```

使用其他的邮件服务商也是类似的方法，但是需要修改对应的 provider ，具体可以参考 imapsync 的[官方文档](https://github.com/imapsync/imapsync/blob/master/README)。
