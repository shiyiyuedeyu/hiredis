这是一份中文文档。  

# HIREDIS
Hiredis是Redis数据库的一个C语言客户端。  

Hiredis仅对Redis提供最基础的支持，但Hiredis优秀的printf-alike API相比其他的略胜一筹，最小的代码量并且不显式绑定每条Redis命令。  

Hiredis只支持binary-safe的Redis协议，如果你想要使用Hiredis，Redis的版本请大于1.2.0。  

# 升级到1.0.0
1.0.0版本是Hiredis的稳定版本。  

# Synchronous API
要使用同步API，只需引入以下几个函数调用：  
```
redisContext *redisConnect(const char *ip, int port);
void *redisCommand(redisContext *c, const char *format, ...);
void freeReplyObject(void *reply);
```

## Connecting
redisConnect用于创建redisContext。redisContext结构具有一个err整数字段(发生错误的时候置为非零状态)，该字段errstr将包含带有错误说明的字符串。使用尝试连接到Redis后，应该检查err字段以判断连接是否成功。  
```
redisContext *c = redisConnect("127.0.0.1", 6379);
if (c == NULL || c->err) {
    if (c) {
        printf("Error: %s\n", c->errstr);
        // handle error
    } else {
        printf("Can't allocate redis context\n");
    }
}
```
注意：redisContext是非线程安全的。  

## Sending commands
有好几种给Redis传递命令的方法。  
第一种是redisCommand，这个函数与printf格式差不多。  
```
reply = redisCommand(context, "SET foo bar");
```
%s和printf类似
```
reply = redisCommand(context, "SET foo %s", value);
```
当你需要传递二进制安全的字符串时，可以利用%b，配合string的指针，还需要string的size_t的length。  
```
reply = redisCommand(context, "SET foo %b", value, (size_t) valuelen);
```
你还可以：  
```
reply = redisCommand(context, "SET key:%s %s", nyid, value);
```

## Using replies
redusCommand成功执行命令后，返回值将保留在reply中。发生错误时，返回值为NULL并且err字段会改变。一旦错误返回，这个context就不能继续使用了，你需要重建一个新的连接。  
redisCommand的标准回复是redisReply类型，type被用来告知收到了哪种回复。  
**REDIS_REPLY_STATUS**：状态字符串的用reply-str，字符串长度用reply-len。  
**REDIS_REPLY_ERROR**：错误字符串的访问方式同REDIS_REPLY_STATUS。  
**REDIS_REPLY_INTEGER**：可以使用reply->integer类型的字段访问整数值 long long。  
**REDIS_REPLY_NIL**：该命令回复了一个nil对象。没有要访问的数据。  
**REDIS_REPLY_STRING**：可以使用来访问回复的值reply->str。可以使用访问此字符串的长度reply->len。  
**REDIS_REPLY_ARRAY**：多批量回复。多批量回复中的元素数存储在中 reply->elements。多批量回复中的每个元素也是一个redisReply对象，可以通过访问reply->element[..index..]。Redis可能会使用嵌套数组进行回复，但这是完全受支持的。  
应该使用该freeReplyObject()功能释放答复。请注意，此函数将负责释放数组和嵌套数组中包含的子答复对象，因此用户无需释放子答复（这实际上是有害的，并且会破坏内存）。  
注意：异步API是不需要调用freeReplyObject。  

## Cleaning up
要断开连接以及释放context，可以使用：  
```
void redisFree(redisContext* c);
```
此函数立即关闭套接字，然后释放context。

## Sending commands (cont'd)
与redisCommand一样，redisCommandArgv也能发送命令：  
```
void *redisCommandArgv(redisContext *c, int argc, const char **argv, const size_t *argvlen);
```
它需要参数的数量，argc字符串数组argv和参数的长度argvlen。为了方便起见，argvlen可以将设置为NULL，并且函数将strlen(3)在每个参数上使用以确定其长度。显然，当任何参数需要二进制安全时，argvlen都应提供整个长度数组。  
返回值的语义与相同redisCommand。  

## Pipelining
为了解释Hiredis如何支持阻塞连接中的流水线操作，需要了解内部执行流程。  
redisCommand调用该系利中的任何功能时，Hiredis首先根据Redis协议格式化命令。然后将格式化的命令放入上下文的输出缓冲区中。该输出缓冲区是动态的，因此他可以容纳任意数量的命令。将命令放入缓冲区后，redisGetReply被调用，该函数具有以下以下两个执行路径：  
1. 输入缓冲区为非空：  
    * 尝试解析来自输入缓冲区的单个应答并返回
    * 如果无法解析任何答复，请继续执行2
2. 输入缓冲区为空：
    * 将整个输出缓冲区写入套接字
    * 从套接字读取，直到可以解析单个答复  

 ```
void redisAppendCommand(redisContext* c, const char* format, ...);
void redisAppendCommand(redisContext* c, int argc, const char** argv, const size_t* argvlen);
 ```
 redisGetReply可用于接收后续答复。此函数的返回值是REDIS_OK或REDIS_ERR，其中后者表示读取答复时发生错误。  
 ```
redisReply* reply;
redisAppendCommand(context, "SET foo bar");
redisAppendCommand(context, "GET foo");
redisGetReply(context, &reply); // reply for SET
freeReplyObject(reply);
redisGetReply(context, &reply); // reply for GET
freeReplyObject(reply);
 ```
此API也可以用于实现阻塞subcriber: 
```
reply = redisCommand(context, "SUBSCRIBE foo");
freeReplyObject(reply);
while(redisGetReply(context, &reply) == REDIS_OK) {
    // consume message 
    freeReplyObject(reply);
}
```

# Asynchronous API
Hiredis的异步API可以和各种event library一起工作，实例展示如何利用libev和libevent配合Hiredis。  

## Connecting
函数redisAsyncConnect能够和redis生成非阻塞连接。它返回一个新创建的redisAsyncContext结构体的指针。当创建后应该查看err字段判断连接是否出现错误，因为连接是非阻塞的，kernel不会立刻返回即使成功accept了这个连接。  
注意：redisAsyncContext 不是线程安全的  
```
redisAsybcContext* c = redisAsyncConnect("127.0.0.1", 6379);
if (c->err) {
    printf("Error: %s\n", c->errstr);
    // handle error
}
```
异步上下文可以包含一个断开连接回调函数，该连接在断开连接时（由于错误或每个用户请求）而被调用。该函数应具有以下原型：  
```
void(const redisAsyncContext* c, int status);
```
断开连接时，将status参数设置为REDIS_OK用户启动断开连接的时间，或REDIS_ERR由于错误导致断开连接的时间。如果为REDIS_ERR，err 则可以访问上下文中的字段以找出错误原因。  
断开回调触发后，上下文对象被释放。当需要重新连接时，断开回调是这样做的一个好方法。  
每个上下文只能设置一次断开回调。对于后续呼叫，它将返回REDIS_ERR。设置断开连接回调的函数具有以下原型：  
```
int redisAsyncSetDisconnectCallback(redisAsyncContext* ac, redisDisconnectCallback* fn);
```
ac->data可以用于将用户数据传递到此回调，可以对redisConnectCallback执行相同的操作。  

## Sending commands and their callbacks
在异步上下文中，由于事件循环的性质，命令会自动流水线化。因此，与同步API不同，只有一种发送命令的方式。由于命令是异步发送到Redis的，因此发出命令需要一个回调函数，该函数在收到回复时被调用。回复回调应具有以下原型：  
```
void(redisAsyncContext* c, void* reply, void* privdate);
```
privdata参数可用于从命令最初排队执行的位置开始，将任意数据写入回调。  
可用于在异步上下文中发出命令的函数是：  
```
int redisAsyncCommand(
    redisAsyncContext* ac, redisCallbackFn* fn, void* privdata,
    const char* format, ...);
int redisAsyncCommandArgv(
    redisAsyncContext * ac，redisCallbackFn * fn，void * privdata，int argc，const  char ** argv，const  size_t * argvlen);
```
这两个功能都像阻塞对方一样工作。返回值是REDIS_OK命令成功添加到输出缓冲区的时间，REDIS_ERR否则返回。例如：当根据用户请求断开连接时，不得将新命令添加到输出缓冲区，并且REDIS_ERR在调用该redisAsyncCommand系列时会返回该命令。  
如果NULL读取了带有回调的命令的答复，则会立即释放它。当命令的回调为non-时NULL，将在回调之后立即释放内存：答复仅在回调期间有效。  
NULL当上下文遇到错误时，将调用所有待处理的回调并进行回复。  
# Disconnecting
可以使用以下方式终止异步连接：  
```
void redisAsyncDisconnect(redisAsyncContext* ac);
```
调用此函数时，连接不会立即终止。取而代之的是，不再接受新命令，并且仅当所有未决命令已写入套接字，已读取其各自的答复并已执行其各自的回调时，才终止连接。此后，将以REDIS_OK状态执行断开连接回调， 并释放上下文对象。  
