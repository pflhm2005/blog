### DNS

## api
从node源码学习一下dns的获取过程  
根据官方文档，获取DNS主要有两种形式的方式
1. dns.lookup
```
const dns = require('node:dns');

dns.lookup('example.org', (err, address, family) => {
  console.log('address: %j family: IPv%s', address, family);
});
// address: "93.184.216.34" family: IPv4
```
这个方法基本上等同我们平时cmd的ping指令，同时内部会走libuv的api，丢进线程池等待处理。需要注意的是，使用这个不一定会走DNS网络请求，因为其内部依赖操作系统的实现。  
比如说我们在hosts文件中添加一个地址
```
2.3.4.5    test.com
```
这时候用node去寻找test.com的IP地址
```
const dns = require('dns');

dns.lookup('test.com', (err, ip) => {
  console.log(ip); // 2.3.4.5
});
```
我们会直接得到hosts里匹配的地址，有时这个结果并不是我们想要的

如果需要走网络请求，得到准备的结果，有另外一个api可以用

2. dnf.Resolver

我们用这个方式去获取test.com的IP地址
```
const dns = require('dns');
const resolver = new dns.Resolver();

resolver.setServers(['8.8.8.8']);
resolver.resolve4('test.com', (err, ip) => {
  console.log(err, ip); // '67.225.146.248'
});
```
发现此时得到一个准确的IP地址，可以看出用这个方式会严格的走DNS服务器查询

## 源码实现
由于知道了原理，所以内部的实现也很明了，这里简单过一下
### lookup

首先会走JS的模块
```
// lib/dns.js
function lookup(hostname, options, callback) {
  ... 
  const err = cares.getaddrinfo(
    req, toASCII(hostname), family, hints, verbatim
  );
  ... 

  return req;
}
```
这里的cares、req等等都是一些c++实现的工具方法，真正的查询方法是getaddrinfo
```
// src/cares_wrap.cc
void GetAddrInfo(const FunctionCallbackInfo<Value>& args) {
  ...
  int err = req_wrap->Dispatch(uv_getaddrinfo,
                               AfterGetAddrInfo,
                               *hostname,
                               nullptr,
                               &hints);
  ...

  args.GetReturnValue().Set(err); // 等同于JS代码的 return err
}
```
经过一系列的操作，这里把查询逻辑丢到给了libuv，hostname就是我们输入的域名
```
// deps/uv/src/getaddrinfo.cc
int uv_getaddrinfo(uv_loop_t* loop,
                   uv_getaddrinfo_t* req,
                   uv_getaddrinfo_cb getaddrinfo_cb,
                   const char* node,
                   const char* service,
                   const struct addrinfo* hints) {
  ...
  if (getaddrinfo_cb) {
    uv__work_submit(loop,
                    &req->work_req,
                    UV__WORK_SLOW_IO,
                    uv__getaddrinfo_work,
                    uv__getaddrinfo_done);
    return 0;
  } else {
    uv__getaddrinfo_work(&req->work_req);
    uv__getaddrinfo_done(&req->work_req, 0);
    return req->retcode;
  }

error:
  if (req != NULL) {
    uv__free(req->alloc);
    req->alloc = NULL;
  }
  return uv_translate_sys_error(err);
}
```
进入libuv后，如果没有回调会直接走同步逻辑，直接返回结果。但是node里是默认有callback，所以走异步逻辑，将查询操作包装成一个work，推入待处理事件队列，等待轮询。这里的uv__getaddrinfo_work就是具体的处理方法
```
static void uv__getaddrinfo_work(struct uv__work* w) {
  ...
  err = GetAddrInfoW(req->node, req->service, hints, &req->addrinfow);
  req->retcode = uv__getaddrinfo_translate_error(err);
}
```
可以看出，libuv最终调用的是[GetAddrInfoW](https://learn.microsoft.com/en-us/windows/win32/api/ws2tcpip/nf-ws2tcpip-getaddrinfow)，这是一个win32api
