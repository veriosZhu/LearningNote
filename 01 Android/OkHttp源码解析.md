# OkHttp源码解析

## 目录

## 基本用法
``` java
    Request request = new Request.Builder().url("https://baidu.com").get().build();
    OkHttpClient okHttpClient = new OkHttpClient();
    // 异步
    okHttpClient.newCall(request).enqueue(new Callback() {
        @Override
        public void onFailure(Call call, IOException e) {

        }

        @Override
        public void onResponse(Call call, Response response) throws IOException {

        }
    });
    // 同步
    okHttpClient.newCall(request).execute();
```

## 源码分析


## RetryAndFollowUpInterceptor