---
date: 2023-04-24 13:26:00
title: filebeat 配置参考
tags:
  - "filebeat"
draft: false
---

这篇笔记主要介绍 filebeat 配置文件各参数含义和一些配置实例。

<!--more-->

``` bash

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

filebeat 用于收集和处理日志，是 Elastic Beats 开源组件中的关键一员，它通过 yaml 文件来控制具体的日志文件收集行为，这里会将 yaml 配置文件作为探讨的主题。

在 Elastic 官方文件中， Beats 家族被定位为轻量级数据传输器，这里的轻量级主要是和 logstash 做比较，但它们的核心功能都是将源数据进行格式化处理后输出到目的端，所以它们的配置文件就相当于对管道进行配置，主要关心数据进入时和数据输出时的动作。

## 配置 input

在 filebeat 中， input 来源非常丰富，可以是标准输入，具体文件，甚至是容器，网络流，对象存储等。我们这里以最常见的 `log` 为例，也就是将具体文件作为输入来进行基本配置定义。

``` yaml
filebeat.inputs:
- type: log
  paths:
    - /var/log/messages
    - /var/log/*.log
  exclude_files: ['\.gz$']
```

`log` 类型的列表项中 `path` 和 `exclude_files` 可以用来定义需要被 filebeat 收集的文件，默认会收集 `path` 中定义的所有文件路径中的文件， `exclude_files` 用来去掉通配路径中不需要的文件。

### Common options

filebeat 将逻辑上的单条日志当作一个 event 来处理，并使用 [Elastic Common Schema](https://www.elastic.co/guide/en/ecs/current/ecs-base.html
) 来描述 event ，其中具体的日志内容的存储字段将是 `event.message` ，还会带有 `event.@timestamp` ， `event.tags` ， `event.labels` 这些元数据字段，而 inputs Common options 中的某些配置可以直接影响 event 中的字段内容。

``` yaml
filebeat.inputs:
- type: log
  paths:
    - /var/log/messages
    - /var/log/*.log
  exclude_files: ['\.gz$']
  tags: ["json"]
  fields:
    app_id: query_engine_12
  fields_under_root: true
```

`tags` 中的内容会直接出现在 `event.tags` 字段中，可以用于后续处理的重要标识。而 `fields` 和 `fields_under_root` 是关联出现的，如果 `fields_under_root` 为 false ，那么 `fields` 中的所有字段将直接出现在 `event.fields` 字段中，如果 `fields_under_root` 为 true ，那么 `fields` 中的每个子字段会成为 event 的根字段，并且遇到同名字段将以 `fields` 中定义的内容覆盖。

### line 配置

可以通过 `exclude_lines` 和 `include_lines` 来具体定义需要收集的内容，它们可以单独或同时出现，但是在同时出现时 `include_lines` 的优先级永远高于 `exclude_lines` 。

``` yaml
filebeat.inputs:
- type: log
  paths:
    - /var/log/messages
    - /var/log/*.log
  exclude_files: ['\.gz$']
  tags: ["json"]
  fields:
    app_id: query_engine_12
  fields_under_root: true
  exclude_lines: ['^DBG']
  include_lines: ['^ERR', '^WARN']
```

关于两者共存的逻辑，具体相关的代码部分是这样的：

``` go
func (h *Harvester) shouldExportLine(line string) bool {
	if len(h.config.IncludeLines) > 0 {
		if !harvester.MatchAny(h.config.IncludeLines, line) {
			// drop line
			return false
		}
	}
	if len(h.config.ExcludeLines) > 0 {
		if harvester.MatchAny(h.config.ExcludeLines, line) {
			return false
		}
	}

	return true
}
```

### multiline 配置

filebeat 还支持将多行日志聚合的功能，需要通过 multiline 相关配置来定义具体的行为。

``` yaml
filebeat.inputs:
- type: log
  paths:
    - /var/log/messages
    - /var/log/*.log
  exclude_files: ['\.gz$']
  tags: ["json"]
  fields:
    app_id: query_engine_12
  fields_under_root: true
  multiline.pattern: '^\['
  multiline.negate: true
  multiline.match: after
```

`multiline.pattern` 用来定义正则表达式，用来定义需要匹配的行； `multiline.negate` 用来定义使用 `multiline.pattern` 进行匹配或者反选， `multiline.negate` 为 true 则为反选， `multiline.negate` 为 false 则为匹配； `multiline.match` 用来定义匹配值向前或者向后组合，具体值有 `before` 和 `after` 。

它的具体工作模式是这样的：首先使用 `multiline.pattern` 进行逐行比较，根据 `multiline.negate` 判定该行是聚合行中的子行或者是聚合行的开始行，再根据 `multiline.match` 进行组合。

示例中的表达式匹配了使用开头为 `[` 的行，由于它的 `multiline.negate` 为 true ，所以不以 `[` 开头的行被认为是聚合行中的子行，并且根据 `multiline.match` 的 `after` ，每次匹配到 `[` 之后，后续的子行会与它聚合为一行，直到下一符合 `[` 开头的行为止。

### json 配置

`json` 配置可以使 filebeat 读取日志文件时将对应的日志内容解析为 json 格式。

``` yaml
filebeat.inputs:
- type: log
  paths:
    - /var/log/messages
    - /var/log/*.log
  exclude_files: ['\.gz$']
  tags: ["json"]
  fields:
    app_id: query_engine_12
  fields_under_root: true
  exclude_lines: ['^DBG']
  include_lines: ['^ERR', '^WARN']
  json:
    keys_under_root: true
    overwrite_keys: true
    add_error_key: true
```

它适用于日志文件本身就是单行 json 格式的场景，因为它默认的解析来源是 `event.message` 字段，不过这个来源可以通过 `message_key` 进行修改。其中参数 `keys_under_root` 和 `overwrite_keys` 的定义类似于 `fields_under_root` 的行为，如果成功解析日志，字段都会被添加到 event 中，且同名字段时也是采取覆盖操作。而 `add_error_key` 则会在解析日志失败时添加 `event.error` 字段并在其中加入报错信息。解析成功后，不会再出现 `event.message` 字段。

### 具体行为配置

filebeat 可以通过某些配置项来定义对日志文件的具体操作细节：

* `scan_frequency` ：扫描通配路径中文件的时间频率，它会监控是否有新文件和已有文件变化，默认值为 10s 。
* `tail_files` ：是否从文件末尾开始读取文件，默认值为 false ，会从头开始读取文件。
* `symlinks` ：是否读取符号链接文件，默认值为 false ，不读取符号链接文件。
* `ignore_older` ：忽略经过某段时间未被修改的文件，实际上是通过文件的修改时间来区分的，默认值为 0 ，即禁用这个功能，使用的话则应该使用 1h ， 5m ， 10s 这种类型的字符串。

在这部分配置中，还有两类重要配置 `clean_*` 和 `close_*` 定义了 filebeat 具体的工作行为，它们涉及到了一部分 filebeat 的工作原理。

我们可以简单认为 filebeat 会周期性地扫描我们定义的文件，每个文件都会启动 Harvester 来进行处理，同时 Harvester 会将一些中间信息记录到 registry 中，它可以认为是 filebeat 进行作业时的中间目录，在 Linux 系统中的默认路径是 `/var/lib/filebeat/registry` 。如果只是处理某个文件的 Harvester 关闭，当这个文件有新变动，会启动新的 Harvester 并根据 registry 中的信息对文件进行处理，比较关键的值就是文件读取的偏移量，而如果 registry 中的关于某个文件的数据已经清除，那么 Harvester 会把它当成全新文件来处理。

`clean_*` 定义了 registry 清理行为的相关内容，而 `close_*` 定义了 Harvester 关闭行为的相关内容。

`close_*` 有如下几个具体配置项：

* `close_eof` ：当文件读取完毕时，是否关闭 Harvester ，默认为 false 。
* `close_inactive` ：当文件读取完毕时， Harvester 并不会马上关闭，而是等待一段时间，如果超过这段时间仍然没有新的数据，则关闭 Harvester ，默认为 5m 。
* `close_timeout` ： Harvester 超时时长，当 Harvester 工作时间超出这个值则关闭 Harvester ，默认为 0 ，也就是禁用超时功能。
* `close_renamed` ：当文件被重命名时，是否关闭 Harvester ，默认为 false 。
* `close_removed` ：当文件被删除时，是否关闭 Harvester ，默认为 true 。

`clean_*` 有如下几个具体配置项：

* `clean_inactive` ：当文件没有产生新的数据时，会等待一段时间，如果超过这段时间仍然没有新的数据，则删除 registry 中相关信息，默认为 0 ，也就是禁用主动清理功能。
* `clean_removed` ：当文件被删除时，是否删除 registry 中相关信息，默认为 true 。

在实际使用中来看， `close_*` 这些配置项比较宽容，因为 `scan_frequency` 的存在，在文件变动时会自动启动新的 Harvester ，所以 Harvester 的关闭行为是可接受的。

在官方文档中也有相关应用的记录，当处于需要近实时发送日志行的场景下，可以通过调整 `close_inactive` ，这样就可以一直由当前 Harvester 处理文件。

但是 `clean_*` 的使用要比较小心，比如在日志写入间隔时间长的场景下，使用不恰当的 `clean_inactive` 和 `clean_removed` 可能会发生日志从头读取的情况，而且 `clean_inactive` 必须大于 `ignore_older + scan_frequency` ，否则也会出现日志从头读取的情况。

## 配置 processor

processor 可以用于处理 event 中的数据，它可以配置在各个 input 中独享，也可以在 inputs 之外配置并由所有 input 共享。 filebeat 会优先执行独享的 processors ，再执行共享配置的 processors 。

``` yaml
# input 独享 processors
filebeat.inputs:
- type: log
  paths:
    - /var/log/messages
    - /var/log/*.log
  exclude_files: ['\.gz$']
  tags: ["json"]
  fields:
    app_id: query_engine_12
  fields_under_root: true
  exclude_lines: ['^DBG']
  include_lines: ['^ERR', '^WARN']
  json:
    keys_under_root: true
    overwrite_keys: true
    add_error_key: true
  processors:
    - add_fields:
        target: project
        fields:
          name: myproject
          id: '574734885120952459'
---
# inputs 共享 processors
filebeat.inputs:
- type: log
  paths:
    - /var/log/messages
    - /var/log/*.log
  exclude_files: ['\.gz$']
  tags: ["json"]
  fields:
    app_id: query_engine_12
  fields_under_root: true
  exclude_lines: ['^DBG']
  include_lines: ['^ERR', '^WARN']
  json:
    keys_under_root: true
    overwrite_keys: true
    add_error_key: true

processors:
  - add_fields:
      target: project
      fields:
        name: myproject
        id: '574734885120952459'
```

关于 processors 的运行顺序逻辑，具体相关的代码部分是这样的：

``` go
type Processors struct {
	List []beat.Processor
	log  *logp.Logger
}

func (procs *Processors) Run(event *beat.Event) (*beat.Event, error) {
	var err error
	for _, p := range procs.List {
		event, err = p.Run(event)
		if err != nil {
			return event, fmt.Errorf("failed applying processor %v: %w", p, err)
		}
		if event == nil {
			// Drop.
			return nil, nil
		}
	}
	return event, nil
}
```

以下会记录一些使用频率比较高的 processor 。 

### add_fields

`add_fields` 用来新增 event 中的字段，如果 `target` 为 `""` 则将 `fields` 内容添加到 event 的根字段，否则将添加 `event.$target` 的字段， `fields` 作为这个字段的内容。这个 processor 在具体行为上和 input 的 `fields` 接近，但 `add_fields` 使用更加灵活。

``` yaml
processors:
  - add_fields:
      target: project
      fields:
        name: myproject
  - add_fields:
      target: ""
      fields:
        name: myproject
```

### drop_fields

`drop_fields` 用来丢弃 event 中的某些字段，一般使用会配合 processor 的条件语句，符合条件才执行这个动作，如果不使用条件判断，则默认都会执行这个丢弃动作，这时 `ignore_missing` 比较重要，可以设置为 true 来屏蔽字段不存在的报错。

``` yaml
processors:
  - drop_fields:
      when：
        equals:
          status: OK
      ignore_missing: false
      fields: ["field1", "field2"]
```

### drop_event

`drop_event` 用来将整个 event 丢弃，这样它不会进行后续处理，可以用来屏蔽一些不需要的数据内容。

``` yaml
processors:
  - drop_event:
      when：
        or:
        - equals:
            http_request_uri: '/health'
        - regexp:
            http_user_agent: 'kube-probe'
```

### convert

`convert` 用来转换字段的数据类型，比较常用的是字符类型和数值类型的相互转换。其中 `ignore_missing` 用来忽略无法正确找到 `from` 来源的错误， `fail_on_error` 用来忽略转换过程中发生的错误，比如格式错误等。

``` yaml
processors:
  - convert:
      fields:
        - {from: "src_ip", to: "source.ip", type: "ip"}
        - {from: "src_port", to: "source.port", type: "integer"}
      ignore_missing: true
      fail_on_error: false
```

### dissect

`dissect` 可以通过给定匹配式截取出需要的数据并且拆分到对应的字段中，可以用于非标准 json 格式日志的信息格式化。其中 `target_prefix` 与 `add_fields` 中的 `target` 类似， `ignore_failure` 和 `overwrite_keys` 也同 `fields` 中错误处理和字段覆写用法相似。

``` yaml
# log_path: /var/log/info.log
# log_message: "789 - App02 - Database is will be restarted in 5 minutes"
processors:
  - dissect:
      tokenizer: "/var/log/%{level}.log"
      target_prefix: ""
      field: "log.file.path"
      ignore_failure: false
      overwrite_keys: false
  - dissect:
      tokenizer: '"%{detail.pid|integer} - %{detail.name} - %{detail.status}"'
      target_prefix: ""
      field: "message"
      ignore_failure: false
      overwrite_keys: false

# final:
# {
#   "level": "info",
#   "detail": {
#     "pid": 789,
#     "name": "App02",
#     "status": "Database is will be restarted in 5 minutes"
#   }
# }
```

`dissect` 执行之后，原有的 `field` 还会保留，如果不再需要这个字段，需要显式调用 `drop_fields` 这个 processor 。

### script

`script` 可以使用 javascript 来处理 event ，已有的内置函数可以参考[官方文档](https://www.elastic.co/guide/en/beats/filebeat/current/processor-script.html)。

``` yaml
# log_path: /var/log/info.log
# log_message: "789 - App02 - Database is will be restarted in 5 minutes"
processors:
  - dissect:
      tokenizer: "/var/log/%{level}.log"
      target_prefix: ""
      field: "log.file.path"
      ignore_failure: false
      overwrite_keys: false
  - dissect:
      tokenizer: '"%{detail.pid|integer} - %{detail.name} - %{detail.status}"'
      target_prefix: ""
      field: "message"
      ignore_failure: false
      overwrite_keys: false
  - script:
      lang: javascript
      source: >
        function process(event) {
          var level = event.Get("level");
          var name = event.Get("detail.name");
          var final_name = name + "-" + level;
          event.Tag("javascript");
          event.Put("index", final_name);
        }
```

## 配置 output

在输入数据经过处理后，我们还需要配置输出端，指明最终的数据流向。

当我们处在调试阶段时，可以直接输出到控制台，如果使用 `systemd` 托管，那么使用 `systemctl status filebeat` 或者 `journalctl -fu filebeat` 就可以看到对应的输出。

``` yaml
output.console:
  pretty: true
```

在生产环境下，可以直接从 filebeat 把数据输出到 ElasticSearch ，也可以使用 Kafka 这类消息中间件做数据中转，再通过其他方式去消费。

``` yaml
# log_path: /var/log/info.log
# log_message: "789 - App02 - Database is will be restarted in 5 minutes"
processors:
  - dissect:
      tokenizer: "/var/log/%{level}.log"
      target_prefix: ""
      field: "log.file.path"
      ignore_failure: false
      overwrite_keys: false
  - dissect:
      tokenizer: '"%{detail.pid|integer} - %{detail.name} - %{detail.status}"'
      target_prefix: ""
      field: "message"
      ignore_failure: false
      overwrite_keys: false
  - script:
      lang: javascript
      source: >
        function process(event) {
          var level = event.Get("level");
          var name = event.Get("detail.name");
          var final_name = level + "-" + name ;
          event.Tag("javascript");
          event.Put("index", final_name);
        }

output.elasticsearch:
  hosts: ["https://localhost:9200"]
  username: "filebeat"
  password: "filebeat_password"
  indices:
    - index: "%{[index]}-%{+yyyy.MM.dd}"
      when.equals:
        level: "info"
    - index: "error-%{[detail.name]}-%{+yyyy.MM.dd}"
      when.contains:
        status: "ERR"
    - index: "default-%{+yyyy.MM.dd}"

output.kafka:
  hosts: ["localhost:9092"]
  required_acks: 1
  compression: gzip
  max_message_bytes: 1000000
  partition.round_robin:
    reachable_only: false
  topics:
    - topic: "%{[index]}"
      when.equals:
        level: "info"
    - topic: "error-%{[detail.name]}"
      when.contains:
        status: "ERR"
    - topic: "default"
```

两者的配置方式非常接近，但应该根据实际情况来选择输出，在日志量巨大的情况下还是输出到 Kafka 比较合适，在 Kafka 配置中需要注意的是 `max_message_bytes` 的默认值为 1M ，它应该和 Kafka 的 `message.max.bytes` 设置一致，而且在日志体过大的时候应该重新调整，否则可能会丢失日志。
