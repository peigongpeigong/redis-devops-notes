## 客户端

### 客户端通信协议
- 客户端与服务端之间的通信协议是在TCP的基础上构建的
- Redis制定了RESP(REdis Serialization Protocal)实现客户端与服务端的正常交互，
  这种协议简单高效，既能够被机器解析，又容易被人类识别

##### 发送命令格式
RESP规定一条命令格式如下，CRLF代表\r\n
```
*<参数数量>CRLF
$<参数1字节数量> CRLF
<参数1> CRLF
...
$<参数N字节数量> CRLF
<参数N> CRLF
```
格式化后为
```
*<参数数量>\r\n$<参数1字节数>\r\n<参数1>\r\n$<参数N字节数>\r\n<参数N>\r\n
```
##### 返回结果格式
返回结果类型分为五种
- 状态回复：在RESP中第一个字节为"+"
- 错误回复：在RESP中第一个字节为"-"
- 整数回复：在RESP中第一个字节为"："
- 字符串回复：在RESP中第一个字节为"$"
- 多条字符串回复：在RESP中第一个字节为"*"

### Jedis
见代码

#### Jedis连接池
见代码

本章目前只了解了Jedis的常规用法，其他内容后补

