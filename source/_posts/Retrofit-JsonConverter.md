---
title: Retrofit JsonConverter
date: 2017-10-21 23:18:35
tags: Android
categories: Android
---
(by 蔡老板)
## 问题概述

笔者在开发过程中临时遇到一个本来仅有web端的项目临时增加Android端，导致后端在出接口时并未考虑Android端的json数据的解析，导致接口是这样的。。。。<!--more-->


* 正确请求

``` json
{
    "code": 0,
    "data": {
        "user": {
            "id": 145,
            "name": "abtion",
            "school_id": 1,
            "sex": "男",
            "add_on": null,
            "status": 1,
            "role": "student",
            "created_at": "2017-08-05 18:26:31",
            "updated_at": "2017-08-19 12:41:50"
        }
    }
}
```

* 错误请求

``` json
{
    "code": 20002,
    "data": "Password Wrong"
}
```

这也就是说在请求正确时服务端返回的数据中data是在java中的一个对象，而错误时却变成了String，这就导致了错误的请求在解析json时抛出异常导致请求失败，而且抛出的异常是无法拿到错误码和错误信息的。

## 问题分析

我们该如何解决这个问题呢，经过思考，方法有三：

* 呼叫可爱的后端老哥改接口，将错误信息改由``message``字段输出
* 给``okhttp``添加拦截器，在``retrofit``解析json前解析json数据并存储。
* 自定义``Gson``响应体变换器和响应变换工厂，在请求错误时抛出异常并保存错误码和错误信息。

由于该项目已经上线，再改接口无异于痴人说梦，因加拦截器的效率也不及第三种方法日后再分享，本次采用自定义``Gson``响应体变换器和响应变换工厂的方法来解决。

## 具体解决办法

### 1、切入点

首先请看一张图片

![](http://oum3tk6e0.bkt.clouddn.com/android4_1.png)

我们通常情况下跟图中一样采用的是Gosn工厂变换器，而本次抛出异常的地方就是这个变换器，自定义工厂变换器就可以完美解决我们的问题。

### 2、自定义Gson响应体变换器

``` java
class GsonResponseBodyConverter<T> implements Converter<ResponseBody,T> {
    private final Gson gson;
    private final Type type;

    public GsonResponseBodyConverter(Gson gson, Type type) {
        this.gson = gson;
        this.type = type;
    }


    @Override
    public T convert(ResponseBody value) throws IOException {
        //将返回的json数据储存在String类型的response中
        String response = value.string();
        //将外层的数据解析到APIResponse类型的httpResult中
        APIResponse httpResult = gson.fromJson(response,APIResponse.class);
        //服务端设定0为正确的请求，故在此为判断标准
        if (httpResult.getCode()==0){
            //直接解析，正确请求不会导致json解析异常
            return gson.fromJson(response,type);
        }else {
            //定义错误响应体，并通过抛出自定义异常传递错误码及错误信息
            ErrorResponse errorResponse = gson.fromJson(response,ErrorResponse.class);
            throw new ResultException(errorResponse.getCode(),errorResponse.getData());
        }
    }
}
```

附上APIResponse类，ErrorResponse类和ResultException类

``` java
public class APIResponse<T> extends BaseModel {
    private int code = -2;
    private T data;

    public int getCode() {
        return code;
    }

    public void setCode(int code) {
        this.code = code;
    }

    public T getData() {
        return data;
    }

    public void setData(T data) {
        this.data = data;
    }
}
```
``` java

public class ErrorResponse {
    private int code;
    private String data;

    public ErrorResponse(int code, String data) {
        this.code = code;
        this.data = data;
    }

    public int getCode() {
        return code;
    }

    public void setCode(int code) {
        this.code = code;
    }

    public String getData() {
        return data;
    }

    public void setData(String data) {
        this.data = data;
    }
}

```
``` java

public class ResultException extends IOException {
    private int code;
    private String msg;



    public ResultException(int code, String msg) {
        this.code = code;
        this.msg = msg;
    }

    public int getCode() {
        return code;
    }

    public void setCode(int code) {
        this.code = code;
    }

    public String getMsg() {
        return msg;
    }

    public void setMsg(String msg) {
        this.msg = msg;
    }
}

```
### 3、自定义响应变换工厂

``` java
class ResponseConverterFactory extends Converter.Factory {

    public static ResponseConverterFactory create() {
        return create(new Gson());
    }


    public static ResponseConverterFactory create(Gson gson) {
        return new ResponseConverterFactory(gson);
    }

    private final Gson gson;

    private ResponseConverterFactory(Gson gson) {
        if (gson == null) throw new NullPointerException("gson == null");
        this.gson = gson;
    }

    @Override
    public Converter<ResponseBody, ?> responseBodyConverter(Type type, java.lang.annotation.Annotation[] annotations, Retrofit retrofit) {
        return new GsonResponseBodyConverter<>(gson,type);
    }

    @Override
    public Converter<?, RequestBody> requestBodyConverter(Type type, java.lang.annotation.Annotation[] parameterAnnotations, java.lang.annotation.Annotation[] methodAnnotations, Retrofit retrofit) {
        TypeAdapter<?> adapter = gson.getAdapter(TypeToken.get(type));
        return new GsonRequestBodyConverter<>(gson, adapter);
    }
}
```

### 4、调用自定义的响应变换工厂

在构造Retrofit时在addConverterFactory()方法中传入ResponseConverterFactory.create()就可以了。


``` java
/**
     * 构造Retrofit
     *
     * @return retrofit
     */
    private static Retrofit getRetrofit() {
        if (retrofit == null) {
            retrofit = new Retrofit.Builder()
                    .baseUrl(Config.APP_SERVER_BASE_URL)
                    .addConverterFactory(ResponseConverterFactory.create())
                    .client(getClient())
                    .build();
        }
        return retrofit;
    }
```
    
### 5、在网络请求的onFailure中接收异常信息并进行处理

``` java
@Override
                public void onFailure(Call<APIResponse<LoginResponse>> call, Throwable t) {
                    if (t instanceof ResultException) {
                        ToastUtil.showToast(((ResultException) t).getMsg(), ((ResultException) t).getCode());
                    } else {
                        ToastUtil.showToast("网络请求失败，请稍后再试");
                    }
                }
                
```

到这里就完成了，别忘了Gson的请求体变换器是default限定的。改改限定符就好了。