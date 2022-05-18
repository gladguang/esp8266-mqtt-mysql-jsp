# 温湿度存储


## 项目说明：


#### 1.温湿度传感器数据通过 ESP8266 传输到 EQM-X
#### 2.EQM-X 将数据存储到mySQL数据库
#### 3.网站或应用端程序可以从数据库或者EQM-X读取数据。网站能发送数据到EQM_X再传输给ESP8266，使ESP8266响应相关动作。

## 实现相关


### 1. 数据发送

- 硬件材料
>+ 安信可NodeMCU-12S(ESP8266  CH340芯片) 开发板
>+ 面包板、杜邦线、带通讯功能的Macro USB线、SHTC3数字型温湿度传感器
>


- 软件
> + Arduino软件
>> ##### 所需安装依赖
>> * 安装 ESP8266 开发板 点击 工具 -> 开发板 -> 开发板管理 -> 搜索 ESP8266 -> 点击安装
>> * 安装 PubSub client 库 项目 -> 加载库 -> 管理库... -> 搜索 PubSubClient -> 安装 PubSubClient by Nick O’Leary
> + ComAssistant.exe
>  + NetAssist v4.3.25.exe
> 


#### 实现技术要点

1. 可自主选择ESP8266连接WIFI，且连接WIFI后不会按reset键后恢复。并将WIFI连接信息打印到串口。[^1][^2]

[^1]: 将所连接的WIFI的ip、ssid打印出来
[^2]: 能提示WIFI是否连接成功信息。

代码中[WiFiManager](https://github.com/tzapu/WiFiManager)库须下载导入。有[太极创客汉化版](https://github.com/taichi-maker/WiFiManager)

相关实现代码:
```
#include <ESP8266WiFi.h>          
#include <DNSServer.h>
#include <ESP8266WebServer.h>
#include <WiFiManager.h>         
 
void setup() {
    Serial.begin(115200);       
    // 建立WiFiManager对象
    WiFiManager wifiManager;
    
    // 自动连接WiFi。以下语句的参数是连接ESP8266时的WiFi名称
    wifiManager.autoConnect("ESP8266");
    
    // 如果您希望该WiFi添加密码，可以使用以下语句：
    // wifiManager.autoConnect("AutoConnectAP", "12345678");
    // 以上语句中的12345678是连接AutoConnectAP的密码
    
    // WiFi连接成功后将通过串口监视器输出连接成功信息 
    Serial.println(""); 
    Serial.print("ESP8266 Connected to ");
    Serial.println(WiFi.SSID());              // WiFi名称
    Serial.print("IP address:\t");
    Serial.println(WiFi.localIP());           // IP
}
 
void loop() {}
```
2. 实现ESP8266接收到温湿度传感器数据，且将数据转换成 json 格式。
相关实现代码:

[json数据操作](http://www.taichi-maker.com/homepage/esp8266-nodemcu-iot/iot-c/esp8266-nodemcu-web-client/esp8266-json-parse/)
```
TODO:
```
3. 实现ESP8266通过MQTT协议与EMQ-X服务器的订阅与数据发送
相关实现代码:
```
#include <ESP8266WiFi.h>          
#include <DNSServer.h>
#include <ESP8266WebServer.h>
#include <WiFiManager.h>
#include <PubSubClient.h>
#include <Ticker.h>

const char* mqttServer = "119.91.205.11";
 
// 如以上MQTT服务器无法正常连接，请前往以下页面寻找解决方案
// https://docs.emqx.com/zh/cloud/latest/connect_to_deployments/esp8266.html#%E8%BF%9E%E6%8E%A5
// http://www.taichi-maker.com/public-mqtt-broker/ 


Ticker ticker;
WiFiClient wifiClient;
PubSubClient mqttClient(wifiClient);
 
int count;    // Ticker计数用变量
 
void setup() {
  Serial.begin(9600);
  
  //设置ESP8266工作模式为无线终端模式
  WiFi.mode(WIFI_STA);
  
  // 连接WiFi
  connectWifi();
  
  // 设置MQTT服务器和端口号
  mqttClient.setServer(mqttServer, 1883);
 
  // 连接MQTT服务器
  connectMQTTServer();
 
  // Ticker定时对象
  ticker.attach(1, tickerCount);  
}
 
void loop() { 
  if (mqttClient.connected()) { // 如果开发板成功连接服务器
    // 每隔3秒钟发布一次信息
    if (count >= 3){
      pubMQTTmsg();
      count = 0;
    }    
    // 保持心跳
    mqttClient.loop();
  } else {                  // 如果开发板未能成功连接服务器
    connectMQTTServer();    // 则尝试连接服务器
  }
}
 
void tickerCount(){
  count++;
}
 
void connectMQTTServer(){
  // 根据ESP8266的MAC地址生成客户端ID（避免与其它ESP8266的客户端ID重名）
  String clientId = "esp8266-" + WiFi.macAddress();
 
  // 连接MQTT服务器
  if (mqttClient.connect(clientId.c_str())) { 
    Serial.println("MQTT Server Connected.");
    Serial.println("Server Address: ");
    Serial.println(mqttServer);
    Serial.println("ClientId:");
    Serial.println(clientId);
  } else {
    Serial.print("MQTT Server Connect Failed. Client State:");
    Serial.println(mqttClient.state());
    delay(3000);
  }   
}
 
// 发布信息
void pubMQTTmsg(){
  static int value; // 客户端发布信息用数字
 
  // 建立发布主题。主题名称以Taichi-Maker-为前缀，后面添加设备的MAC地址。
  // 这么做是为确保不同用户进行MQTT信息发布时，ESP8266客户端名称各不相同，
  String topicString = "esp8266/" + WiFi.macAddress();
  char publishTopic[topicString.length() + 1];  
  strcpy(publishTopic, topicString.c_str());
 
  // 建立发布信息。信息内容以Hello World为起始，后面添加发布次数。
  String messageString = "Hello World " + String(value++); 
  char publishMsg[messageString.length() + 1];   
  strcpy(publishMsg, messageString.c_str());
  
  // 实现ESP8266向主题发布信息
  if(mqttClient.publish(publishTopic, publishMsg)){
    Serial.println("Publish Topic:");Serial.println(publishTopic);
    Serial.println("Publish message:");Serial.println(publishMsg);    
  } else {
    Serial.println("Message Publish Failed."); 
  }
}

// ESP8266连接wifi
void connectWifi(){
 
 // 建立WiFiManager对象
    WiFiManager wifiManager;
    
    // 自动连接WiFi。以下语句的参数是连接ESP8266时的WiFi名称
    wifiManager.autoConnect("ESP8266");
    
    // 如果您希望该WiFi添加密码，可以使用以下语句：
    // wifiManager.autoConnect("AutoConnectAP", "12345678");
    // 以上语句中的12345678是连接AutoConnectAP的密码
    
    // WiFi连接成功后将通过串口监视器输出连接成功信息 
    Serial.println(""); 
    Serial.print("ESP8266 Connected to ");
    Serial.println(WiFi.SSID());              // WiFi名称
    Serial.print("IP address:\t");
    Serial.println(WiFi.localIP());           // IP 
}:
```


### 2. 数据存储

- 硬件所需：无
- 软件：
> EMQ X客户端


#### 技术实现要点
1. 通过自带的WebHook。[^3]

[^3]: WebHook处理事件是单向的，仅支持EMQ-X事件推送给Web服务，不关心Web服务返回。借助WebHook可以完成设备在线、上下线记录，订阅与消息存储、消息送达确认等业务。

2. MQTT客户端订阅消息再转存数据库

> 1. 后台再开个超级权限MQTT客户端
> 2. 订阅所需主题
> 3. Qos设置成2。保证只接受到1次。
> 4.将接收到的消息存储到数据库。
> 

相关实现代码:
```
TODO:
```

### 3. 数据库的建立
- 建立mqtt数据库
- 建表。表名为esp_temperature
- 字段约束

```
CREATE DATABASE mqtt;

USE  mqtt;

# 字段约束
TODO：

```

### 4. 网页或客户端显示相关数据

- 硬件所需：无
- 软件：
> IDEA 


#### 技术实现要点：

1. 与数据库交互

相关实现代码:
```
TODO:
```
2. 用jsp显示实时温湿度数据及时间

相关实现代码:
```
TODO:
```
3. 能查询历史数据

相关实现代码:
```
TODO:
```
4.与ESP8266交互。将8266上的小灯开启和熄灭。

相关实现代码:
```
TODO:
```