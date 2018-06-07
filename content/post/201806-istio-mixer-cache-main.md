+++
title = "Istio Mixer Cache工作原理与源码分析(3)－主要代码"

date = 2018-06-06
lastmod = 2018-06-06
draft = true

tags = ["Istio", "Mixer"]
summary = "Mixer Cache的主要实现代码。"
abstract = "Mixer Cache的主要实现代码。"

[header]
image = "headers/dreammesh-architecture-1.jpg"
caption = ""

+++

## 前言

经过前面的基础概念的介绍，我们现在已经可以勾勒出一个mixer cache的实现轮廓，当然实际代码实现时会有很多细节。但是为了方便理解，我们在深入细节之前，先给出一个简化版本，让大家快速了解mixer cache的实现原理。后面的章节我们再逐渐深入。

1. ​


## Check Cache的主流程

对mixer cache的调用在代码 `proxy/src/istio/mixerclient/client_impl.cc` 中的方法Check()中，此处跳过quota cache的内容：

```go
CancelFunc MixerClientImpl::Check(
    const Attributes &attributes,
    const std::vector<::istio::quota_config::Requirement> &quotas,
    TransportCheckFunc transport, CheckDoneFunc on_done) {
	++total_check_calls_;
    
    std::unique_ptr<CheckCache::CheckResult> check_result(
        new CheckCache::CheckResult);
    check_cache_->Check(attributes, check_result.get()); // 在这里调用了CheckCache.Check()方法
    CheckResponseInfo check_response_info;

    check_response_info.is_check_cache_hit = check_result->IsCacheHit();
    check_response_info.response_status = check_result->status();

    // 如果check cache命中，并且结果不OK，则直接结束处理
    if (check_result->IsCacheHit() && !check_result->status().ok()) {
        on_done(check_response_info);
        return nullptr;
    }
    ......
    CheckCache::CheckResult *raw_check_result = check_result.release();
    ......
    // 如果check cache没有命中，则需要发起请求得到response
    // 然后将response加入check cache
    raw_check_result->SetResponse(status, *request_copy, *response);
    ......
}
```

## 保存Check结果到Mixer Cache

当第一次访问或者没有匹配到缓存时（后面再谈如何匹配缓存），这时需要访问mixer，得到处理结果和随response而来的referencedAttributes。

envoy保存mixer结果到缓存的主要步骤是：

1. ​

