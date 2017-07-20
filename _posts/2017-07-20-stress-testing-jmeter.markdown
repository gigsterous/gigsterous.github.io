---
layout: post
title: Stress testing secured application with JMeter
date: 2017-07-20 19:00:00 +0200
author: Martin
description: In this post, I am going to show you how to use JMeter for stress testing a REST application secured with OAuth2.
categories: engineering
tags:
  - Testing
  - JMeter
---

Recently, I found myself in need of stress testing an OAuth2 flow of one of our applications when working at [Berrycloud](https://berrycloud.cz/). What I needed to do is to test the login and logout process (obtaining and revoking the access token) repeatedly.

The desired test should look as follows:
1) Obtain an access token
2) Perform a random request to some secured endpoint
3) Revoke the token
4) Repeat 1.

There are multiple approaches how to write such a test case but there is an excellent tool just for a use case like this - [JMeter](https://jmeter.apache.org/):

> The Apache JMeterapplication is open source software, a 100% pure Java application designed to load test functional behavior and measure performance. It was originally designed for testing Web Applications but has since expanded to other test functions.
> -- Apache JMeter documetation

First of all, simply download JMeter and launch the jar file. A default test plan is already open so let's start modifying it. Right-click on it and create a new **Thread group**. In the thread group configuration, you can specify the number of threads to run this test case, number of iterations, duration/start time etc.


![Thread Group](/assets/2017-07-20-stress-testing-jmeter/jmeter01.png)

The whole test plan is basically a tree where you can attach and nest nodes to specify the workflow. Our next step is creating the first HTTP request for obtaining the access token under the thread group. I will demonstrate the configuration against a localhost environment in this post although, normally, you would probably be stress testing an application in production or staging.

![HTTP Request](/assets/2017-07-20-stress-testing-jmeter/jmeter02.png)

Typically, an `/oauth/token` endpoint is secured with a basic authorization that looks as follows: `Basic <Base64 encoded clientId:clientSecret>`. Additionally, `grant_type` needs to be specified. Let's assume that we are using the `password` grant type with a user called 'john' and password 'secret':

![Request Parameters](/assets/2017-07-20-stress-testing-jmeter/jmeter03.png)

Additionally, headers with basic auth and content type needs to be specified as well. You can add additional configuration to the request by right-clicking it and selecting the **add** option. In this case, we need a **HTTP Header Manager**:

![Request Headers](/assets/2017-07-20-stress-testing-jmeter/jmeter04.png)

If the request is configured correctly, you can now click **run** and execute it. You probably won't see much in JMeter but we can add **View Results Tree** node for checking the reponse to see if the request was successful. Our next step is getting the access token from the JSON response. Let's say that the response body looks as follows:

```json
{
    "access_token":"370592fd-b9f8-452d-816a-4fd5c6b4b8a6",
    "token_type":"bearer",
    "expires_in":43199,
    "scope":"read write"
}
```

JMeter has a post-processor called [JSON Extractor](http://jmeter.apache.org/usermanual/component_reference.html#JSON_Extractor) which is just what we need. It will let us grab the access token and save it to a local variable that could be referenced in the future request. Let's save it with the same name - `access_token`:

![JSON Extractor](/assets/2017-07-20-stress-testing-jmeter/jmeter05.png)

It seems that we are finished with our first request for obtaining the token. Now let's create another HTTP request in a simillar fashion. In this case, I will want to hit a `/principal` endpoint and use the token for getting the secured content. We can reference the token we saved in the previous request with `${access_token}` in the header manager:

![Principal Request](/assets/2017-07-20-stress-testing-jmeter/jmeter06.png)

Lastly, let's revoke the token so our flow is complete. Assuming that we have a `/revoke-token` endpoint, this can be done with additional HTTP request:

![Revoke Token Request](/assets/2017-07-20-stress-testing-jmeter/jmeter07.png)

Everything is ready to run now so if we configured everything correctly, we can set the desired number of threads and iterations and start the stress test. The results could be viewed udner the **View Results Tree** node.

## Summary
Today, we learnt how JMeter could be used for stress testing a certain request flow against a web application. Obviously, this is just a small piece of functionality that is bundled in JMeter and I do recommend reading through the [official documentation](http://jmeter.apache.org/). The capabilities for monitoring and debugging are astonishing.

