---
title: 'Retrofit 自定义Converter.Factory实现直接接收String'
date: 2016-09-09 14:19:19
tags: Android
categories: Android
---

#前言
在使用Retrofit过程中，通过服务器获取的数据，不一定是标准的json数据，当时就想能不能有一种方式，可以把数据直接获取到而不是解析好的数据
<!--more-->

#开始实现
主要是实现MediaType，以及responseBodyConverter和requestBodyConverter方法；
```java
public class ToStringConverterFactory extends Converter.Factory {
	private static final MediaType MEDIA_TYPE = MediaType.parse("text/plain");


	@Override
	public Converter<ResponseBody, ?> responseBodyConverter(Type type, Annotation[] annotations, Retrofit retrofit) {
		if (String.class.equals(type)) {
			return new Converter<ResponseBody, String>() {
				@Override
				public String convert(ResponseBody value) throws IOException {
					return value.string();
				}
			};
		}
		return null;
	}

	@Override public Converter<?, RequestBody> requestBodyConverter(Type type, Annotation[] parameterAnnotations,
			Annotation[] methodAnnotations, Retrofit retrofit) {
		if (String.class.equals(type)) {
			return new Converter<String, RequestBody>() {
				@Override
				public RequestBody convert(String value) throws IOException {
					return RequestBody.create(MEDIA_TYPE, value);
				}
			};
		}
		return null;
	}
}
```
#开始使用
```java
public interface WechatService {

	@GET("/txapi/weixin/wxhot") Call<String> getHotArticleStr(@Header("apikey") String apiKey,
			@Query("num") int num, @Query("rand") int rand, @Query("word") String word, @Query("page") int page,
			@Query("src") String src);
}
```
```java
public void getHotArticleRxString(int num, int rand, String word, int page, String src) {
		Retrofit retrofit = new Retrofit
				.Builder()
				.addConverterFactory(new ToStringConverterFactory())
				.baseUrl(baseurl).build();
		WechatService service = retrofit.create(WechatService.class);
		Call<String> resultObser = service.getHotArticleStr(baiduApiKey, num, rand, word, page, src);
		resultObser.enqueue(new Callback<String>() {
			@Override public void onResponse(Call<String> call, Response<String> response) {
				Log.e("jiangcy", "ToStringConverterFactory : " + response.body().toString());
			}

			@Override public void onFailure(Call<String> call, Throwable t) {

			}
		});
	}
```

[demo 地址](https://github.com/jiangchunyu/RetrofitTest)