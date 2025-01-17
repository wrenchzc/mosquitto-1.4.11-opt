
# 版本说明

本版本基于 https://github.com/houjixin/mosquitto-1.4.11-opt

mosquitto版本基于 1.4.11
原作者增加了查询在线状态，　上下线通知等功能

在此基础上 本仓库
* 集成了认证插件，　添加了基于mysql的说明
* 没有开启TLS的时候，　也生成mosquitto_passwd



# 编译说明

1. 默认不开启TLSa, 修改了mosquitto_passwd.c, 使其支持WITH_TLS:=no
   config.mk 里面
   WITH_TLS:=no

   ```
   make
   sudo make install

   ```

# 配置说明
1. 开启离线消息推送
配置以下选项
```
persistence true
persistence_file mosquitto.db
persistence_location d:/tmp/mosquitto
max_queued_messages 100000
```

2. 配置IP和端口

```
bind_address 0.0.0.0
port 1883
```

3. 配置mysql数据库验证
```
auth_plugin /usr/local/sbin/auth-plug.so
auth_opt_host localhost
auth_opt_port 3306
auth_opt_dbname dbname
auth_opt_user username
auth_opt_pass password
auth_opt_userquery SELECT device_secret FROM lb_device WHERE device_sn = '%s'
auth_opt_superquery SELECT COUNT(*) FROM iot_user WHERE username = '%s' AND is_super = 1
auth_opt_aclquery SELECT topic FROM iot_acl WHERE (device_sn = '%s') AND (rw >= %d)
# auth_opt_anonusername AnonymouS
```

密码使用 mosquitto-auth-plug/np 生成
用户缺省使用LbDevice表
超级用户在　iotUser表中配置 isSuper属性需要设置为　1

需要配置ACL
deviceSn: 设备SN
topic 需要配置的topic, 可以配置多条，　已权限最大的为准
rw 权限　/* 0 无权限　1 只读　2　读写 */





# 在源码上做了3项性能优化，并增加了如下功能：
1. 动态修改keepalive时间；
2. 接收连接上下线通知；
3. 导出在线连接的id和ip到目录：/tmp/online_users
4. 直接查询指定连接ID的在线状态
   
**注意**
如果不启用这些功能向，只需用#注释掉相应配置参数即可

# 新增功能1：
## 功能说明 
向一个topic发送一个时间值，mosquitto将把当前连接的keepalive时间修改为pub过来的时间值；不配置该项意味着不启用该功能；

## 使用说明 
通过参数"topic_change_keepalive"配置一个字符串作为topic，任何一个客户端连接只要向这个topic发送一个大于的时间值，那么该客户端连接
的keepalive时间就被修改为了所发送的时间值；

## 注意 
只能修改自己的（pub消息的那个客户端连接）keepalive时间，无法修改其他连接的时间；

## 配置参数 

topic_change_keepalive client/change/keepalive

# 新增功能2：
## 功能说明 
向指定topic发送上下线通知的消息

## 使用说明 

打开下面两个配置项“topic_notice_online”（对应上线消息）和“topic_notice_offline”（对应下线消息），
并为他们分别设置一个参数，这个设置的参数将被作为一个topic，也可以将这两个topic参数设置成一样，这样，上下线消息都会发送到同一个topic上；
在有连接建立或断开时mosquitto将向这两个topic发送消息，

## 消息格式说明 
mosquitto向下面这两个配置topic发送的上下线消息为JSON字符串，共包含三个字段：
* clientid：当前通知所涉及的连接ID；
* type:连接的状态：1：连接建立；0：连接断开；
* time:当前系统时间，1970年1月1日到现在的时间；
例如：
收到的上下线消息
为如下的json格式：
```
{
	"clientid": "350012",
	"type": "1",
	"time": 1565747719000
}
```

## 配置参数 

```
topic_notice_online client/notice/status/onoffline
topic_notice_offline client/notice/status/onoffline
```

# 新增功能3
## 功能说明 
导出在线连接的id和ip到目录：/tmp/online_users
## 使用说明 
开启配置参数"topic_dump_connection"和"cmd_dump_connection"，为这两个参数各设置一个字符串，其中：
参数topic_dump_connection指定了一个topic，参数cmd_dump_connection指定了一个“命令”，只有向这里配置的topic发送配置的命令
mosquitto才会执行此功能，不配置topic_dump_connection就意味着不启动该功能，一旦启用该功能就必须要同时提供对参数cmd_dump_connection的配置。
## 注意 

1. 这里定义的topic 不能 以$SYS/开头，否则客户端就无法向这个topic发送命令了;
2. 建议这个topic要比较特殊，尽量短，这样效率更高，最好第一个字符跟其他的所有topic都不一样;
3. 为了安全起见，参数cmd_dump_connection不能设置为空，如果命令匹配，mosquitto才会执行此功能；
4. 为了尽量降低对mosquitto的影响，该命令一分钟只能执行一次；
5. 下面这两个参数要么全配，要么全部不配
## 配置参数 

topic_dump_connection client/connection/dump
cmd_dump_connection hello-jason

# 新增功能4：
## 功能说明 
查询指定连接ID是否在线，返回JSON格式字符串，JSON格式与新增功能2一致：1：在线；0：不在线
## 使用方式 
开启下面配置，该配置将指定一个topic，任何一个客户端只要向这个topic发布一个连接ID（即pub过来的消息内容就是要查询的连接ID），mosquitto
就会给这个当前pub消息的客户端回复一条消息，查询的客户端无需订阅任何topic，只要向这里配置的topic发布连接ID，就能收到mosquitto发布过来的查询结果。
## 配置参数 

topic_query_conn_status client/query/connection
