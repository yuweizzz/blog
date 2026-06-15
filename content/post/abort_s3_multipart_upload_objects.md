---
date: 2026-04-02 10:00:00
title: 清理 S3 multipart upload 对象
tags:
  - "s3"
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

直接在 aws 控制台执行即可。

```bash
# 检查当前存储桶所有的 multipart upload 对象
aws s3api list-multipart-uploads --bucket "$bucket" --query Uploads[*].[Key,UploadId,Initiated,Initiator.DisplayName] --output text

# 删除单个 multipart upload 对象
aws s3api abort-multipart-upload --bucket "$bucket" --key "$key" --upload-id "$uploadid"

# 确认无误后可以使用批量删除
aws s3api list-multipart-uploads --bucket "$bucket" --query Uploads[*].[Key,UploadId] --output text | while read key uploadid
do
  aws s3api abort-multipart-upload --bucket "$bucket" --key "$key" --upload-id "$uploadid"
done
```

更好的办法是设置生命周期规则，可以自行定义 multipart upload 对象的过期删除时间。
