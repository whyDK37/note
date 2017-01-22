
> [原文地址](https://bourgeois.me/rest/)，有不正确的地方欢迎指正，谢谢。

REST APIs are a very common topic nowaday; they are part of almost every web application. A simple, consistent and pragmatic interface is a mandatory thing; it will be much easier for others to use your API. Even if these practices may look common to your eye, I often see people that don't really respect them. That's why I decided to write a post about it.
Here are some best practices to keep in mind while designing a RESTful API.
Disclamer: these best practices are what I think is good, based from my past expriences. If you think otherwise, feel free to send me an email so we can have a discussion about it.

REST APIs 现今是通用的一般规则，它几乎是所有web应用的一部分，一个简单，一致，实用的接口是强制的。你的api会对使用者来说会更容易。虽然这些实践在你看来是常见的，我经常看到人们不遵守，这就是为什么我决定写一篇关于它的文章。
## Version your API 你的API要有版本

API versions should be mandatory. This way, you will be futureproof as the API changes through time. One way is to do is to pass the API version in the URL (/api/v1/...).

One other neat trick is to use the Accept HTTP header to pass the verison desired. [Github does that](https://developer.github.com/v3/media/#request-specific-version).

Using versions will allow you to change the API structure without breaking compatibility with older clients.

API 版本应该是强制的。这样，你的程序将不会随着时间而过时。一种方式是在 URL 中传递特定的API 版本(/api/v1/...)。一种更优雅的方式是使用HTTP Accept header 来传递期望的版本。[Github 就是这样做的](https://developer.github.com/v3/media/#request-specific-version)。

## Use nouns, not verbs 使用名词，而不是动词

One thing I often see is people using verbs instead of nouns in their resources name. Here are some bad examples :

我经常看到人们在资源名中使用动词来代替名词，下面是一些不好的例子：

- /getProducts
- /listOrders
- /retreiveClientByOrder?orderId=1

For a clean and concise structure, you should always use nouns.
Moreover, a good use of HTTP methods will allow you to remove actions from your resources names.

为了结构简洁，你应该始终使用名词。而且，有效利用 HTTP method 允许你把资源中的动作移除。

- GET /products : will return the list of all products
- POST /products : will add a product to the collection
- GET /products/4 : will retreive product #4
- PATCH/PUT /products/4 : will update product #4

It is much cleaner that way.

这种方式更简洁。

## Use the plural form 使用复数形式

In my opinion, it is not a really good idea to mix singular and plural forms in a single resource naming; it can quickly become confusing and non consistent.

依我看来，在相同的资源上混合使用单数和复数不是个好主意。它会很快会变得混乱，不一致。

Use /artists instead of /artist, even for the show/delete/update actions

使用/artists 代替 /artist，即使对于 show/delete/update 命令。

## GET and HEAD calls should always be safe GET和HEAD应该永远是安全的

[RFC2616](https://www.w3.org/Protocols/rfc2616/rfc2616-sec9.html) cleary states that HEAD and GET methods should always be safe to call (in other words, the state should not be altered).

[RFC2616](https://www.w3.org/Protocols/rfc2616/rfc2616-sec9.html) 声明，HEAD 和 GET 的应该是一直可以安全调用的（换句话，状态不应该改变）。

Here is a bad example: GET /deleteProduct?id=1

这是一个不好的例子：GET /deleteProduct?id=1

Imagine a search engine indexing that page...

想象搜索引擎把那个页面编入索引中，（搜索引擎是不会知道你分页参数是什么的）

> 本章节的内容不是很理解

## Use nested resources 使用嵌套资源

If you want to get a sub collection (collection of an other one), use nested routing for a cleaner design. For instance, if you want to get a list of all albums of a particular artist, you would want to use GET /artists/8/albums.

如果你想获取一个自集（另一个集合），使用嵌套路由使设计简单。如：如果你想获取特定艺术家的所有专辑，你将使用 GET /artists/8/albums。
## Paging

Returning a very large resultset over HTTP is not a very good idea neither. You will eventually run into performance issues as serializing a large JSON may quickly become expensive.

通过http返回一个大的结果集不是一个好主意，你将最终遭遇性能问题，如序列化一个大json可能会非常耗费性能。

An option to get around that would be to paginate your results. Facebook, Twitter, Github, etc. does that. It is much more efficient to make more calls that takes little time to complete, than a big one that is very slow to execute.

将结果分页是一个逃避这个问题选项。 Facebook, Twitter, Github, 等等，就是这么做的。进行更多的在短时间内成功的调用，比一个大的执行非常慢的调用更有效率。

Also, if you are using pagination, one good way to indicate the next and previous pages links is through the Link HTTP header. [Github does that too](https://developer.github.com/guides/traversing-with-pagination/).

另外，如果你使用分页，一个好的方式是在HTTP header 中指定上一个和下一页。[Github 就是这么做的](https://developer.github.com/guides/traversing-with-pagination/).

## Use proper HTTP status codes 使用适当的 HTTP 状态码

Always use proper HTTP status codes when returning content (for both successful and error requests). Here a quick list of non common codes that you may want to use in your application.

当返回内容的时候，始终使用适当的HTTP状态码（包括成功和失败的请求），这里你可能要在您的应用程序中使用的非常见代码快速列表。
### Success codes 成功码

- 201 Created should be used when creating content (INSERT),
- 202 Accepted should be used when a request is queued for background processing (async tasks),
- 204 No Content should be used when the request was properly executed but no content was returned (a good example would be when you delete something).

### Client error codes 客户端错误码

- 400 Bad Request should be used when there was an error while processing the request payload (malformed JSON, for instance).
- 401 Unauthorized should be used when a request is not authenticiated (wrong access token, or username or password).
- 403 Forbidden should be used when the request is successfully authenticiated (see 401), but the action was forbidden.
- 406 Not Acceptable should be used when the requested format is not available (for instance, when requesting an XML resource from a JSON only server).
- 410 Gone Should be returned when the requested resource is permenantely deleted and will never be available again.
- 422 Unprocesable entity Could be used when there was a validation error while creating an object.

A more complete list of status codes can be found in [RFC2616](https://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html).

更完整的状态码列表可以在[RFC2616](https://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html)找到。

## Always return a consistent error payload 始终返回一致的错误信息
When an exception is raised, you should always return a consistent payload describing the error. This way, it will be easier for other to parse the error message (the structure will always be the same, whatever the error is).

当引发异常，你应该始终返回一致的信息表述哪个错误，这种方式，会是他人更容易解析错误信息（不管是什么错误，结构要始终相同）。

Here one I often use in my web applications. It is clear, simple and self descriptive.

这个是我经常在我的web 应用中使用。他清楚，简单，自解释。

```
HTTP/1.1 401 Unauthorized
{
    "status": "Unauthorized",
    "message": "No access token provided.",
    "request_id": "594600f4-7eec-47ca-8012-02e7b89859ce"
}
```