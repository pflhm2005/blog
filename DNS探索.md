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
可以看出，libuv最终调用的是[GetAddrInfoW](https://learn.microsoft.com/en-us/windows/win32/api/ws2tcpip/nf-ws2tcpip-getaddrinfow)，这是一个win32api，来源于"ws2tcpip.h"，属于windows内部api，可以在winSDK里面找到，具体的实现就暂时不看了

### Resolver

这个类的声明在JS代码里，但是所有方法的实现都在C++，只看resolve4这个方法  
首先来看一下类的定义
```
// lib/dns.js
const { Resolver, resolveMap } = createResolverClass(resolver);
function createResolverClass(resolver) {
  const resolveMap = ObjectCreate(null);
  class Resolver extends ResolverBase {}
  Resolver.prototype.resolveAny = resolveMap.ANY = resolver('queryAny');
  Resolver.prototype.resolve4 = resolveMap.A = resolver('queryA');
  ...

  return {
    resolveMap,
    Resolver
  };
}
```
这里的Resolver就是dns.Resolver，其中有一个写法还挺有意思，可以说是最简化的单例
```
function lazyBinding() {
  binding ??= internalBinding('cares_wrap');
  return binding;
}
```
这里的??=是只有binding为null或undefined时才会对其赋值，所以只会赋值一次，后续的调用会直接返回  
源码比较绕，这里直接看resolve4的实现
```
// 这里是个模板函数resolve4 => queryA => QueryAWrap => ATraits
template <class Wrap>
static void Query(const FunctionCallbackInfo<Value>& args) {
  ...

  auto wrap = std::make_unique<Wrap>(channel, req_wrap_obj);

  node::Utf8Value name(env->isolate(), string);
  channel->ModifyActivityQueryCount(1);
  int err = wrap->Send(*name);
  ...

  args.GetReturnValue().Set(err); // 相当于JS代码的return err
}
```
这里的核心就是Send方法，name就是我们传的域名
```
void AresQuery(const char* name, int dnsclass, int type) {
  ...

  ares_query(
      channel_->cares_channel(),
      name,
      dnsclass,
      type,
      Callback,
      MakeCallbackPointer());
}
```
走的是方法ares_query，这个方法来源于外部依赖[c-ares](https://c-ares.org/)，一个异步的DNS工具  
多嘴一句，这个方法所在文件名是cares_wrap，里面很多方法在node各处都能见到，之前一直搞不懂什么意思，看到这个工具名才明白是对第三方插件的封装，吐了啊……  
简述一下工具内部实现，C++源码没啥好看的，看总结：
1. 生成一个query对象来管理DNS请求，包含了uid、待请求域名、callback等属性
2. 处理传入的DNS服务器IP，判断请求方式是否是tcp(未指定请求方式且请求体size大于512强制为tcp)
3. 调用winsock2.h的socket方法发送DNS请求
4. 处理返回结果