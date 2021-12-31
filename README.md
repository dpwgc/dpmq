# DPMQ.消息队列

## 基于Golang Gin整合Gorilla WebSocket实现的消息队列

`Golang` `Gin` `Gorilla` `WebSocket` `MQ`

***

### 实现功能

* 订阅推送：将单条消息通过WebSocket结合go并发推送到多个与消息topic主题相同的消费者客户端。

* 流量削峰：用Golang channel的缓冲区充当队列存储大量积压的消息，按指定时间间隔批量推送缓冲区中的消息。

* 消息推送失败重试机制，定时重推过期消息。

* 支持数据持久化。

* 定期清除早期消息。

***

### 主要部分

##### 生产者消息接收 `producer.go`

* 生产者客户端通过HTTP请求发送消息到消息队列，消息被写入消息通道。

```
//把消息写入消息通道
messageChan <- message
```

##### 消费者消息推送 `consumer.go`

* 消费者客户端通过WebSocket连接到消息队列。

* 消息队列从消息通道中读取出消息后，通过for循环结合go协程并发遍历消费者客户端集合，判断该消费者是否与消息同属于一个主题，如果是，则将消息通过WebSocket推送给该客户端。

```
//读取消息通道中的消息
message := <-messageChan

...

//遍历消费者客户端集合
for key, consumer := range consumers {

	//多协程并发推送消息
	go func(key string,consumer *websocket.Conn) {
	    ...
	    //发送消息到消费者客户端
	    err := consumer.WriteJSON(message)
	    ...
	}
}
```

##### 消息通道 `mq.go`

* 使用golang的通道充当队列，通道的缓冲空间大小决定了消息队列的容量。

```
//消息通道，用于存放待消费的消息(有缓冲区)
var messageChan = make(chan models.Message, messageChanBuffer)
```

##### 控制台 `console`

* 用于获取消费者客户端列表及消息队列配置信息。`consoleApi.go`

```
//获取全部消费者客户端集合 GetConsumers
GET http://localhost:port/Console/GetConsumers

//获取消息队列详细配置 GetConfig
GET http://localhost:port/Console/GetConfig

//获取指定状态的消息记录列表
GET http://localhost:port/Console/GetMessageList

//获取所有状态的消息记录列表
GET http://localhost:port/Console/GetAllMessageList
```

* 用于获取消费者客户端列表、消息队列配置信息、各状态消息列表。`consolePage.go`

```
//前端网页 - 控制台页面
GET http://localhost:port/Console/
```


***

### 连接方式

#### 路由 `router.go`

```
//生产者接口（http post请求，用于接收生产者客户端发送的消息）
r.POST("/Producer/Send",servers.ProducerSend)

//消费者连接（WebSocket连接，用于推送消息到消费者客户端）
r.GET("/Consumers/Conn/:topic/:consumerId", servers.ConsumersConn)
```

#### 访问路径

###### 生产者客户端发送消息到消息队列

* POST `http://localhost:port/Producer/Send`

```
POST请求参数：
Header:
secretKey     //访问密钥

Body:
messageData   //消息内容  类型：json string
topic         //所属主题  类型：string（不能包含符号“|”）
```

```
消息发送成功，返回数据（msg为messageCode）：
{
    "code": 0,
    "msg": "d9a624ffa17a6c51fcb2381686dd335161b7252d"
}

消息发送失败，返回数据（msg为报错信息）：
{
    "code": -1,
    "msg": "topic不能包含字符“|”"
}
```

###### 消息队列通过WebSocket连接推送消息给消费者客户端

* WebSocket `ws://localhost:port/ConsumersConn/{topic}/{consumersId}`

```
WebSocket链接中的参数：
topic        //主题名称（不能包含符号“|”）
consumersId  //消费者客户端Id（不能重复，不能包含符号“|”）
```

```
推送给消费者客户端的消息格式
{
    "MessageCode":"8c01b728ef82ba754a63e61daa43e83c61b744c7",
    "MessageData":"hello",
    "Topic":"test_topic",
    "CreateTime":"1640975470",
    "Status":0
}
```

***

### 项目结构

##### config 配置类

* application.yaml `项目配置文件`

* config.go `项目配置文件加载`

##### model 实体类

* model.go `消息模板`

##### persistent 持久化

* fileRW.go `文件读写`

* persData.go `持久化到硬盘`

* recovery.go `数据恢复`

##### router 路由

* router.go `路由配置`

##### server 服务层

* console `控制台`

    * consoleApi.go `控制台接口`

    * consolePage.go `控制台页面`

* producer.go `生产者消息接收`

* consumer.go `消费者消息推送`

* mq.go `消息队列`

* log.go `日志记录`

* rePush `消息重推与过期消息清理`

##### utils 工具类

* createCode.go `消息标识码生成`

* localTime.go `获取本地时间`

* md5Sign.go `md5加密`

* toTimestamp.go `日期字符串转时间戳`

##### main.go 主函数

***

### 配置说明

* application.yaml

```
server:
  # 运行端口号
  port: 8011

mq:
  # 生产者、控制台访问密钥（放在请求头部）
  secretKey: test_secret_key
  # 消息通道的缓冲空间大小（消息队列的容量）
  messageChanBuffer: 10000
  # 推送消息的速度（{pushMessagesSpeed}秒/一批消息）
  pushMessagesSpeed: 1
  # 单批次推送的消息数量
  sendCount: 5
  # 消息推送失败后的重试次数
  sendRetryCount: 3
  # 持久化文件存储路径
  persistentPath: ./data.csv
  # 是否进行持久化（1：是。0：否）
  isPersistent: 1
  # 数据恢复策略（0：清空本地数据，不进行数据恢复。1：将本地数据恢复到内存）
  recoveryStrategy: 1
  # 两次持久化的间隔时间（单位：秒）
  persistentTime: 2
  # 重新推送消息的速度（{rePushSpeed}秒/一批消息）
  rePushSpeed: 5
  # 消息清理时间阈值（当消息存在{clearTime}秒后，删除该消息）
  clearTime: 259200
```

***

### 打包方式

* 填写application.yaml内的配置。

* 运行项目：

```
（1）GoLand直接运行main.go(调试)
```

```
（2）打包成exe运行(windows部署)

  GoLand终端cd到项目根目录，执行go build main.go命令，生成main.exe文件
```

```
（3）打包成二进制文件运行(linux部署)

  cmd终端cd到项目根目录，依次执行下列命令：
  SET CGO_ENABLED=0
  SET GOOS=linux
  SET GOARCH=amd64
  go build main.go
  生成main文件
```

***

### 部署方式

```
在Windows上部署

/dpmq                     # 文件根目录
    DPMQ.exe              # 打包后的exe文件
    /config               # 配置目录
        application.yaml  # 配置文件
    /log                  # 日志目录
    /view                 # 前端目录
        Index.html        # 控制台页面
    data.csv              # 持久化文件
```

```
在Linux上部署

/dpmq                     # 文件根目录
    DPMQ                  # 打包后的二进制文件(程序后台执行:setsid ./DPMQ)
    /config               # 配置目录
        application.yaml  # 配置文件
    /log                  # 日志目录
    /view                 # 前端目录
        Index.html        # 控制台页面
    data.csv              # 持久化文件
```

***