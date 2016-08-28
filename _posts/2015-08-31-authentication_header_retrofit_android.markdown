---
layout: post
title: Request-dependant headers in Retrofit
date: '2015-08-31'
cover_image: '/content/images/2015/8/rest.png'
tags:
- retrofit
- rest
- android
summary: Solving the issue of adding a request-dependant header in Retrofit, by exploring Retrofit 2.0.0-beta1 and OkHttp 2.2+
---
## Background

### What is Retrofit?
[Retrofit](http://square.github.io/retrofit/) is a "A type-safe HTTP client for Android and Java" made by [Square](http://square.github.io/) with a couple of widely used libraries:
 1. [Picasso](http://square.github.io/picasso/) "A powerful image downloading and caching library for Android"
 2. [OkHttp](http://square.github.io/okhttp/) "An HTTP & SPDY client for Android and Java applications"
 3. [Otto](http://square.github.io/otto/) "An enhanced event bus with emphasis on Android support"
 4. [Dagger](http://square.github.io/dagger/) "A fast dependency injector for Android and Java"

Note: I'm not affiliated with Square, I'm just a big fan of what [@JakeWharton](https://github.com/JakeWharton) is doing.

### How to use Retrofit?
I'm not getting into details on basic how-tos as these are covered in way too many blogs, I recommend checking this [post by Robin Chutaux](http://blog.robinchutaux.com/blog/a-smart-way-to-use-retrofit/) for Hello World and more advanced usage of Retrofit.


## Requirement
While working on a project I was required to add an Authentication header param to each request for validation on the backend.

### Basic solution
Normal way to toggle this would to follow Robin's SessionRequestInterceptor

{% highlight Java %}

public class SessionRequestInterceptor implements RequestInterceptor {
    @Override
    public void intercept(RequestFacade request)
    {
      if (AndroidApplication.getAuthenticationToken() != null)
          request.addHeader("Authentication", AndroidApplication.getAuthenticationToken());
    }
}

{% endhighlight %}

And don't forget adding the interceptor to your `RestAdapter`
{% highlight Java %}

        RestAdapter restAdapter = new RestAdapter.Builder()
                .setLogLevel(RestAdapter.LogLevel.FULL)
                .setEndpoint(BASE_URL)
                .setConverter(new GsonConverter(gson))
                .setRequestInterceptor(new SessionRequestInterceptor()) // This is the important line ;)
                .build();

{% endhighlight %}

### Problem
The requirement wasn't about a simple token to be generally used if existed, it was about a token that involved **hashing the request URL, Type, Body and Timestamp** before adding the result to the request header (to protect against [Replay Attacks](https://en.wikipedia.org/wiki/Replay_attack))

So basically it required access to the following:
1. Request Type
2. Request Url
3. Request Body (if any)

#### Ways to achieve that

##### 1. Using [@Header](http://square.github.io/retrofit/javadoc/retrofit/http/Header.html)

If it was a specific request that required this header, `@Header` would be the way to go.
{% highlight Java %}

@GET("/user")
Call<User> getUser(@Header("Authorization") String authorization);

{% endhighlight %}
Which will be called like this
{% highlight Java %}

GitHubService service = restAdapter.create(GitHubService.class);
String authorizationHash = SecurityManager.getHashedRequest(GET, BASE_URL+"/user", null);
service.getUser(authorizationHash);

{% endhighlight %}

Of course doing this for each request is a killer overhead and would render using Retrofit useless, as all the needed data are encapsulated in the interface and can't be retrieved by code. **(Hello Copy and Paste mistakes)**

##### 2. Using Global variables to save request parts

Same annoyance as using `@Header` except that this solution will also render Async requests not valid

##### 3. Using [Retrofit 2.0.0-beta1](http://square.github.io/retrofit/)

1. Note that it's a **beta** release so no guarantees, **YOU MUST FINISH [THIS PRESENTATION by @JakeWharton](https://speakerdeck.com/jakewharton/simple-http-with-retrofit-2-droidcon-nyc-2015) BEFORE GOING FORWARD**
2. Migrating to Retrofit 2.0.0-beta1 is a **_big_** step, as most of your networking code might need some changes (Check [Calls in API Declaration](http://square.github.io/retrofit/#api-declaration))
3. Retrofit 2.0.0 itself isn't the solution, [OkHttp 2.2+](https://github.com/square/okhttp) is. Due to the introduction of [Interceptors](https://github.com/square/okhttp/wiki/Interceptors) concept
3. Retrofit 2.0.0 doesn't have logging built-in yet, so you're gonna have to cook your own Logger (Will be covered here later, [gist](https://gist.github.com/mSobhy90/7613df0db31d0ad77952#file-logginginterceptor-java))

### Solution details

Source code can be found in this [gist](https://gist.github.com/mSobhy90/7613df0db31d0ad77952)

These 3 lines of code are basically the solution
{% highlight Java %}

OkHttpClient client = new OkHttpClient();
client.interceptors().add(new SecurityOfficerInterceptor());

Retrofit restAdapter = new Retrofit.Builder()
                .client(client)
                .baseUrl(URLs.getServerUrl())
                .build();


{% endhighlight %}

## Miscellaneous info

#### 1. Why the switch to Retrofit 2.0.0-beta1?
Because pre-2.0.0 retrofit was using a different version of OkHttp, where the OkHttp client didn't support interceptors.

#### 2. Logging isn't yet supported in Retrofit 2.0.0-beta1
Use [LoggingInterceptor](https://gist.github.com/mSobhy90/7613df0db31d0ad77952#file-logginginterceptor-java) along with your new `OkHttpClient` to Log request & response details (update it to fit your logging needs)

**Important note:** while updating `LoggingInterceptor` make sure to not consume the response body before it reaches the server, follow the copy mechanism in the current class

#### 3. Default converters are no longer part of Retrofit library
To use Gson default converter for example you must include
```
    compile 'com.squareup.retrofit:converter-gson:2.0.0-beta1'
```
in your `build.gradle`

And then add the converter to your `Retrofit` object as follows:
{% highlight Java %}

Retrofit restAdapter = new Retrofit.Builder()
                .client(client)
                .baseUrl(URLs.getServerUrl())
                .addConverterFactory(GsonConverterFactory.create(gson())) // Important line
                .build();

{% endhighlight %}



---

_Cover photo: [REST](http://developer.decarta.com/img/hero_rest_API.png)_
