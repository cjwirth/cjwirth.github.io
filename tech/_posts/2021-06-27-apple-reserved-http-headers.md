---
layout: post
title: Why Not to use Apple's Reserved HTTP Headers
---

When developing an iOS app, there is a good chance you’ll have to make network requests to communicate with a backend server. A common way to communicate authorization information to the server is by using the `Authorization` HTTP header. However, you may run into problems if you try using this and other headers directly.

<!--excerpt-->

According to [Apple’s documentation](https://developer.apple.com/documentation/foundation/nsurlrequest#1776617) certain headers are handled by the URL loading system, and the system may ignore, overwrite, or just not send any value that you set.

The reserved headers are:
* Content-Length
* Authorization
* Connection
* Host
* Proxy-Authenticate
* Proxy-Authorization
* WWW-Authenticate

A lot of these make sense for the system to take care if. For example, I don’t want to have to calculate the `Content-Length`, and the `Host` can be determined from the URL. Others like `Authorization`, I might want to customize to gain access to certain API features.

Let’s take a look at how iOS treats the authorization header.

## Detecting Headers on the Backend
In order to see what headers a backend server will actually receive, we need to set up a server and inspect the headers we get. This is easy enough to do in python:

```python
import os
from flask import Flask, request

app = Flask(__name__)

@app.route('/')
def hello():
    print("hello headers:")
    print(request.headers)
    return "hello"

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

With this, we have a simple server that will print out the headers it receives when a request is made to it.

On the iOS side, we can pull up a playground and make requests to this server after adding some headers:

```swift
let session = URLSession(configuration: .ephemeral)
var request = URLRequest(url: URL(string: "http://127.0.0.1:5000/")!)
request.setValue("normal-auth-header", forHTTPHeaderField: "Authorization")
request.setValue("new-auth-header", forHTTPHeaderField: "X-My-Authorization")
session.dataTask(with: request) { data, response, error in
    print("done")
}.resume()

print("go")
```

Here we are making a request to our new server with the `Authorization` header set to “normal-auth-header” and a custom `X-My-Authorization` header set to “new-auth-header”.

As expected, the server prints out the logs we were expecting, including both of the headers we sent:

```
hello headers:
Host: 127.0.0.1:5000
X-My-Authorization: new-auth-header
Connection: keep-alive
Accept: */*
User-Agent: MyApp/1 CFNetwork/1237 Darwin/20.5.0
Authorization: normal-auth-header
Accept-Language: en-us
Accept-Encoding: gzip, deflate


127.0.0.1 - - [27/Jun/2021 14:21:14] "GET / HTTP/1.1" 200 -
```

At this point, all seems normal, and you may think that the `Authorization` header is safe to use.

## Losing the Authorization Header
Let’s say that your backend API server needs to change the endpoints, but still wants to support existing apps. You might create a redirect for existing endpoints to send them to their new counterparts.

We can add a method to our backend server that does just this, and redirects to out main endpoint that outputs the headers:

```python
# Add “redirect” to the imports
from flask import Flask,redirect,request

# …

@app.route('/redirect')
def redirect_to_root():
    print("redirect headers:")
    print(request.headers)
    return redirect("http://127.0.0.1:5000/", code=301)
```

Here we are returning a 301 HTTP status code and telling the caller to go to the root endpoint instead. Conveniently, this redirect is handled for us under the hood, so we don’t have to worry about the it happening at all.

We can keep our existing code on iOS, but change the URL to point to the redirect endpoint instead:

```swift
let session = URLSession(configuration: .ephemeral)
var request = URLRequest(url: URL(string: "http://127.0.0.1:5000/redirect")!)
request.setValue("normal-auth-header", forHTTPHeaderField: "Authorization")
request.setValue("new-auth-header", forHTTPHeaderField: "X-My-Authorization")
session.dataTask(with: request) { data, response, error in
    print("done")
}.resume()

print("go")
```

This time, we see something a little bit different in what our backend server prints out:

```shell
redirect headers:
Host: 127.0.0.1:5000
X-My-Authorization: new-auth-header
Connection: keep-alive
Accept: */*
User-Agent: MyApp/1 CFNetwork/1237 Darwin/20.5.0
Authorization: normal-auth-header
Accept-Language: en-us
Accept-Encoding: gzip, deflate


127.0.0.1 - - [27/Jun/2021 14:24:43] "GET /redirect HTTP/1.1" 301 -
hello headers:
Host: 127.0.0.1:5000
X-My-Authorization: new-auth-header
Accept: */*
User-Agent: MyApp/1 CFNetwork/1237 Darwin/20.5.0
Accept-Language: en-us
Accept-Encoding: gzip, deflate
Connection: keep-alive


127.0.0.1 - - [27/Jun/2021 14:24:43] "GET / HTTP/1.1" 200 -
```

Notice how this time the `Authorization` header has disappeared by the time it gets to the root endpoint. Our custom `X-My-Authorization` header is still there, so it’s not the case that all headers are being stripped, it’s quite specifically the `Authorization` header that has not been passed along.

I’m not entirely sure why it is being stripped, as that part is not documented. It does say that certain may be ignored or overwritten, and this one scenario where it may happen unexpectedly.

## What to do Instead

As we have seen up above, our custom `X-My-Authorization` header was preserved through the redirect. My recommendation would be to use these custom headers if possible, such as if you have control over the backend server that you are connecting to.

You could also try changing the way your server handles authorization credentials. Maybe you could send them somewhere else, like inside the cookie instead.

If you’re connecting to a server that you don’t have control over, then you will have to intercept the redirect response, and recreate the new request, preserving all the headers that you want to use. Something something I’m not sure how to do that in modern `URLSession` world, that seems to be an old `NSURLConnection` thing to do.

## Conclusion

Although `Authorization` is a common way of sending authorization credentials to a backend server, we have seen one way that the header can just be lost in transit. This could result in some confusing bugs in your app where users get logged out or can’t access information because the server doesn’t know who they are. Instead, if we refrain from using HTTP headers that are documented as reserved by Apple, then we can see that our headers make it though, and those bugs shouldn’t happen.