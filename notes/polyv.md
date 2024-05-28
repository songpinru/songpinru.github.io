Polyv集成设计文档

# 背景

Iparllay集成polyv

# 需求描述

https://www.tapd.cn/54503349/prong/stories/view/1154503349001007135

# 功能划分

![polyv对接设计](polyv.assets/polyv%E5%AF%B9%E6%8E%A5%E8%AE%BE%E8%AE%A1.jpg)

# 模块设计

## workflow

目前workflow都是以platId和openId（还有企微的unionId）为基础创建的，节点信息存储以及上下游的接口对接都是基于这两个Id来实现的，但是polyv是不属于微信生态的，我们无法直接获得platId和openId，这里我们需要变更为我们的客户中台的businessId和contactId

为了兼容openId和contactId，不

### 工作流创建

集成polyv无法直接使用微信生态的openId和platId（替换为我们的contactId和BusinessId），为了区分，需要标记工作流是哪种

mysql增加字段

QAChannel_Business_PPE.qa_workflow表增加字段

```sql
ALTER TABLE QAChannel_Business_PPE.qa_workflow ADD version tinyint DEFAULT 0 COMMENT '0:使用openId或unionId,1:使用contactId' AFTER created_by ; 
```

接口变动：

POST /workflow/manage

```json
{
  "id": 2008,
  "platId": "K7BA1A629G",
  "businessId": 2,
  "type": 1,
  "title": "工作流演示",
  "description": "hello world",
  "target": "{\"type\":\"all\"}",
  "workflowElements": [],
  "statistics": null,
  "oneOff": false,
  "enable": true,
  "status": 1,
  "createdBy": null,
  "version":1,                                    --增加version,0：旧数据源，1：新数据源
  "createdDate": null,
  "lastUpdate": null,
  "properties": null
}
```

### Element存储

Event新增2个Property：

name:envent

value:polyv

```json
{
    "elementId": 8654,
    "id": 7399,
    "name": "event",
    "operation": "in",
    "value": "polyv"
}
```

name：eventValue

value：polyv-{eventType}-{channelId}

* 提交报名观看事件 submit
* 登记报名观看事件 apply
* 进入直播间事件 play

```json
{
    "elementId": 8654,
    "id": 7399,
    "name": "eventValue",
    "operation": "in",
    "value": "polyv-submit-123456789"
}
```

Rule新增5个Property：

* polyv_rule_playduration_{channelId}
  
  * 观看时长，类型int64，单位ms

* polyv_rule_livestatus_{channelId}
  
  * 直播状态，类型string
  * unStart：未开始
    live：直播中
    end：已结束
    waiting：等待中
    playback：回放中

* polyv_rule_playtype_{channelId}
  
  * 观看类型，类型string
  
  * vod：观看回放
    
    live：观看直播

* polyv_rule_isplaylive_{channelId}
  
  * 是否观看直播，类型string
  
  * yes：是
    
    no：否

* polyv_rule_playcount_{channelId}
  
  * 观看次数，类型int

```json
{
    "elementId": 8654,
    "id": 7399,
    "name": "polyv_rule_playduration_123456789",
    "operation": "greaterThan",
    "value": "1000"
}
```

### 新增一个Action

新增一个空动作 countdown来实现倒计时功能

```json
{
    "id": 75615,
    "displayId": "block-1641285715788",
    "workflowId": 2008,
    "type": "Action",
    "metaString": "",
    "incomings": "[\"path-1\"]",
    "outgoings": "[\"path-1641285715788\"]",
    "properties": [],
    "extensions": "{\"action\":\"countdown\",\"actionValue\":{,\"countdown\":\"polyv_rule_starttime_123456789\",\"countdownValue\":\"1000\",\"delay\":0,\"within\":null,\"rule\":null,\"ruleValue\":null,\"repeat\":null}"
  }
```

extension增加：

* countdown
  * 倒计时依赖的属性，string
  * 目前只有polyv_rule_starttime_{channelId}
  * 和delay是冲突的，如果有countdown，delay就不会生效
* countdownValue
  * 倒计时时长，单位ms，int64

### Business接口

Business服务增加一个接口，用来获取channel信息，用户设置polyv事件和属性时选择使用哪个channel

```json
GET /polyv/channels/businessId/{businessId}

response:
{
  "success": true,
  "error": "",
  "data": [
    {
      "id": 794,
      "businessId": 170,
      "channelId": "2701602",
      "name": "脸脸测试直播",
      "liveStatus": "playback",
      "watchUrl": "https://live.polyv.cn/watch/2701602",
      "startTime": 1638171000000,
      "status": 1,
      "createTime": 1645583059000,
      "updateTime": 1645583059000
    },
    {
      "id": 795,
      "businessId": 170,
      "channelId": "2639783",
      "name": "测试",
      "liveStatus": "unStart",
      "watchUrl": "https://live.polyv.cn/watch/2639783",
      "startTime": 1635395288000,
      "status": 1,
      "createTime": 1645583059000,
      "updateTime": 1645583059000
    }
  ]
}
```

### 事件接口

POST  /rest/polyv/event

```json
{
  "event":"submit",
  "channel_id":"2707858",
  "business_id":170,
  "contact_id":"23dd2ab909454a469e4093b5e86cf87b"
}
```
