# 日志规范

## 一、信息日志格式

[公共字段][公共字段],用Key Value对表示的日志内容

具体格式如下：

```
[operation][message],{key1=value1,key2=value2,key3=value3}
```

公共字段：

|名称|类型|含义|
|---|----|---|
|logtime|timestamp_text|日志记录的时间|
|operation|string|日志类型|

## 二、异常日志规范

具体示例如下：

```
[exceptiontype][message],{key1=value1,key2=value2,key3=value3}{errorStack:xxx}
```

## 三、约定
- operation：必须的，各业务方自己定义一下operation
- message：可选，目前中英文无规定，各业务方可自行选择





