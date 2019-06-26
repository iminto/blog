---
title: Filebeat7自定义index的一个坑
date: 2019-06-26 13:21:52
tags: [Java]
---
我用的filebeat7来收集日志发给Elastic search，版本是7.1.1，对应的elasticsearch版本和其相同。

默认的，filebeat生成的索引名字是filebeat-7.1.1-2019.06.24这种，不利于区分不同的业务，需要自定义索引，看了下官方文档，
是这么写的
```
indexedit
The index name to write events to. The default is "filebeat-%{[agent.version]}-%{+yyyy.MM.dd}" (for example, "filebeat-7.2.0-2019-06-26"). If you change this setting, you also need to configure the setup.template.name and setup.template.pattern options (see Load the Elasticsearch index template).

If you are using the pre-built Kibana dashboards, you also need to set the setup.dashboards.index option (see Load the Kibana dashboards).

You can set the index dynamically by using a format string to access any event field. For example, this configuration uses a custom field, fields.log_type, to set the index:

output.elasticsearch:
  hosts: ["http://localhost:9200"]
  index: "%{[fields.log_type]}-%{[agent.version]}-%{+yyyy.MM.dd}" 
```
重点提到了还需要修改setup.template.name和setup.template.pattern，于是我配置如下：
```yaml
output.elasticsearch:
    hosts: ["127.0.0.1:9200"]
    index: "ngerr-%{[agent.version]}-%{+yyyy.MM.dd}"
setup.template.name: "ngerr"
setup.template.pattern: "ngerr-*"
```
结果发现无论如何都不生效，找了很多文章都说配置这几个地方就行，包括google也搜不到结果。我试了不同的配置，甚至把setup.template配置调了位置，还是徒劳，控制台永远都是输出如下
```
2019-06-26T13:11:20.287+0800    INFO    pipeline/output.go:95   Connecting to backoff(elasticsearch(http://127.0.0.1:9200))
2019-06-26T13:11:20.294+0800    INFO    elasticsearch/client.go:734     Attempting to connect to Elasticsearch version 7.1.1
2019-06-26T13:11:20.379+0800    INFO    [index-management]      idxmgmt/std.go:223      Auto ILM enable success.
2019-06-26T13:11:20.380+0800    INFO    [index-management.ilm]  ilm/std.go:134  do not generate ilm policy: exists=true, overwrite=false
2019-06-26T13:11:20.380+0800    INFO    [index-management]      idxmgmt/std.go:238      ILM policy successfully loaded.
2019-06-26T13:11:20.380+0800    INFO    [index-management]      idxmgmt/std.go:361      Set setup.template.name to '{filebeat-7.1.1 {now/d}-000001}' as ILM is enabled.
2019-06-26T13:11:20.380+0800    INFO    [index-management]      idxmgmt/std.go:366      Set setup.template.pattern to 'filebeat-7.1.1-*' as ILM is enabled.
2019-06-26T13:11:20.380+0800    INFO    [index-management]      idxmgmt/std.go:400      Set settings.index.lifecycle.rollover_alias in template to {filebeat-7.1.1 {now/d}-000001} as ILM is enabled.
2019-06-26T13:11:20.380+0800    INFO    [index-management]      idxmgmt/std.go:404      Set settings.index.lifecycle.name in template to {filebeat-7.1.1 map[policy:{"phases":{"hot":{"actions":{"rollover":{"max_age":"30d","max_size":"50gb"}}}}}]} as ILM is enabled.
2019-06-26T13:11:20.383+0800    INFO    template/load.go:129    Template already exists and will not be overwritten.
2019-06-26T13:11:20.383+0800    INFO    [index-management]      idxmgmt/std.go:272      Loaded index template.
2019-06-26T13:11:20.524+0800    INFO    [index-management]      idxmgmt/std.go:283      Write alias successfully generated.
```
{filebeat-7.1.1 {now/d}-000001} 这个名字总是会覆盖我自己的配置。反复尝试，觉得是 ILM 这个东西在作梗，于是试着搜索了下“filebeat ILM is enabled”，发现了这个[issue](https://github.com/elastic/beats/issues/11866) ，有不少人踩坑了。提出issue的人也指出了文档没有说清楚。

指向了这个官方文档：https://www.elastic.co/guide/en/beats/filebeat/current/ilm.html

原来

> Starting with version 7.0, Filebeat uses index lifecycle management by default when it connects to a cluster that supports lifecycle management. Filebeat loads the default policy automatically and applies it to any indices created by Filebeat.

可惜的是filebeat的配置项那里一直没有说清楚。网上由于大多数人用的都是很保守的配置和较老的版本，所以很难搜索到类似的问题，我基本上是头几个踩坑的了。

加上这个配置就好了：

```yaml
setup.ilm.enabled: false
```

希望能帮到踩坑的人，我已经在这个问题上浪费了三四个小时了。

