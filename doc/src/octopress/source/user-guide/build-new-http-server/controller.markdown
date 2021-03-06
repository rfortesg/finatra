---
layout: user_guide
title: "Add an HTTP Controller"
comments: false
sharing: false
footer: true
---

<ol class="breadcrumb">
  <li><a href="/finatra/user-guide">User Guide</a></li>
  <li><a href="/finatra/user-guide/build-new-http-server">Building a New HTTP Server</a></li>
  <li class="active">Add a Controller</li>
</ol>

## HTTP Controller Basics
===============================

We now want to add the following controller to the [server definition](/finatra/user-guide/build-new-http-server#server-definition):

```scala
import ExampleService
import com.twitter.finagle.http.Request
import com.twitter.finatra.http.Controller
import javax.inject.Inject

class ExampleController @Inject()(
  exampleService: ExampleService
) extends Controller {

  get("/ping") { request: Request =>
    "pong"
  }

  get("/name") { request: Request =>
    response.ok.body("Bob")
  }

  post("/foo") { request: Request =>
    exampleService.do(request)
    "bar"
  }
}
```
<div></div>

The server can now be defined with the controller as follows:

```scala
import DoEverythingModule
import ExampleController
import com.twitter.finatra.http.routing.HttpRouter
import com.twitter.finatra.http.{Controller, HttpServer}

object ExampleServerMain extends ExampleServer

class ExampleServer extends HttpServer {

  override val modules = Seq(
    DoEverythingModule)

  override def configureHttp(router: HttpRouter): Unit = {
    router.
      add[ExampleController]
  }
}
```
<div></div>

Here we are adding *by type* allowing the framework to handle class instantiation.

## <a class="anchor" name="controllers-and-routing" href="#controllers-and-routing">Controllers and Routing</a>
===============================

Routes are defined in a [Sinatra](http://www.sinatrarb.com/)-style syntax which consists of an HTTP method, a URL matching pattern and an associated callback function. The callback function can accept either a [`c.t.finagle.http.Request`](https://github.com/twitter/finagle/blob/develop/finagle-http/src/main/scala/com/twitter/finagle/http/Request.scala) or a custom case-class that declaratively represents the request you wish to accept. In addition, the callback can return any type that can be converted into a [`c.t.finagle.http.Response`](https://github.com/twitter/finagle/blob/develop/finagle-http/src/main/scala/com/twitter/finagle/http/Response.scala).

When Finatra receives an HTTP request, it will scan all registered controllers **in the order they are added** and dispatch the request to the **first matching** route starting from the top of each controller then invoking the matching route's associated callback function. That is, routes are matched in the order they are added to the [`c.t.finata.http.routing.HttpRouter`](https://github.com/twitter/finatra/blob/develop/http/src/main/scala/com/twitter/finatra/http/routing/HttpRouter.scala). Thus if you are creating routes overlapping URIs it is recommended to list the routes in order starting with the "most specific" to the least specific.

In general, however, it is recommended to that you follow [REST](https://en.wikipedia.org/wiki/Representational_state_transfer) conventions if possible, e.g., when deciding which routes to group into a particular controller, group routes related to a single resource into one controller. The per-route stating provided by Finatra in the [`c.t.finatra.http.filters.StatsFilter`](https://github.com/twitter/finatra/blob/develop/http/src/main/scala/com/twitter/finatra/http/filters/StatsFilter.scala) works best when this convention is followed.

```scala
class GroupsController extends Controller {
  get("/groups/:id") { ... }

  post("/groups") { ... }

  delete("/groups/:id") { ... }
}
```
<div></div>

yields the following stats:

```
route/groups_id/GET/...
route/groups/POST/...
route/groups_id/DELETE/...
```
<div></div>

Alternatively, each route can be assigned a name which will then be used to create stat names.

```scala
class GroupsController extends Controller {
  get("/groups/:id", name = "group_by_id") { ... }

  post("/groups", name = "create_group") { ... }

  delete("/groups/:id", name = "delete_group") { ... }
}
```
<div></div>

which will yield the stats:

```
route/group_by_id/GET/...
route/create_group/POST/...
route/delete_group/DELETE/...
```
<div></div>

### Route Matching Patterns:

#### Named Parameters

Route patterns may include named parameters:

```scala
get("/users/:id") { request: Request =>
  "You looked up " + request.params("id")
}
```
<div></div>

**Note:** *Query params and route params are both stored in the "params" field of the request.* If a route parameter and a query parameter have the same name, the route parameter always wins. Therefore, you should ensure your route parameter names do not collide with any query parameter name that you plan to read.

#### Wildcard Parameter

Routes can also contain the wildcard pattern. The wildcard can only appear once at the end of a pattern and it will capture all text in its place. For example,

```scala
get("/files/:*") { request: Request =>
  request.params("*")
}
```
<div></div>

For a `GET` of `/files/abc/123/foo.txt` the endpoint will return `abc/123/foo.txt`

#### Regular Expressions

Regular expressions are no longer allowed in string defined paths (since v2).

#### <a class="anchor" name="route-prefix" href="#route-prefix">Route Prefixes</a>

Finatra provides a simple DSL for adding a common prefix to a set of routes within a Controller. For instance, if you have a group of routes within a controller that should all have a common prefix you can define them by making use of the `c.t.finatra.http.RouteDSL#prefix` function available in any subclass of `c.t.finatra.http.Controller`, e.g.,

```
class MyController extends Controller {

  // regular route
  get("/foo") { request: Request =>
    "Hello, world!"
  }

  // set of prefixed routes
  prefix("/2") {
    get("/foo") { request: Request =>
      "Hello, world!"
    }

    post("/bar") { request: Request =>
      response.ok
    }
  }
}
```

This definition would produce the following routes:

```
GET     /foo
GET     /2/foo
POST    /2/bar
```

The input to the `c.t.finatra.http.RouteDSL#prefix` function is a String and how you determine the value of that String is entirely up to you. You could choose to hard code the value like in the above example, or inject it as a parameter to the Controller (by using a [flag](/finatra/user-guide/getting-started#flags) or a [Binding Annotation](/finatra/user-guide/getting-started/#binding-annotations) that looks for a bound String type in the object graph which would allow you provide it in any manner appropriate for your use case).

E.g.,

```
class MyController @Inject()(
  @Flag("api.version.prefix") apiVersionPrefix: String, // value from a "api.version.prefix" flag
  @VersionPrefix otherVersionPrefix otherApiVersionPrefix: String // value from a String bound with annotation: @VersionPrefix
) extends Controller {
  ...

  prefix(apiVersionPrefix) {
    get("/foo") { request: Request =>
      ...
    }
  }

  prefix(otherVersionPrefix) {
    get("/bar") { request: Request =>
      ...
    }
  }
```

##### Things to keep in mind:
- Routes are always added to the [`c.t.finata.http.routing.HttpRouter`](https://github.com/twitter/finatra/blob/develop/http/src/main/scala/com/twitter/finatra/http/routing/HttpRouter.scala)` [in the order defined in the Controller](/finatra/user-guide/build-new-http-server/controller.html#controllers-and-routing) (and are scanned in this order as well), this remains true even when defined within a `prefix` block. I.e., the `prefix` is merely a convenience for adding a common prefix to a set of routes. You should still be aware of the total order in which your routes are defined in a Controller.
- You can use the `c.t.finatra.http.RouteDSL#prefix` function multiple times in a Controller with the same or different values.

#### <a class="anchor" name="admin-paths" href="#admin-index">Admin Paths</a>

All [TwitterServer](http://twitter.github.io/twitter-server/)-based servers have an [HTTP Admin Interface](https://twitter.github.io/twitter-server/Features.html#admin-http-interface) which includes a variety of tools for diagnostics, profiling, and more. This admin interface should not be exposed outside your DMZ. Any route path starting with `/admin/` or `/admin/finatra/` will be included on the server's admin interface (accessible via the server's admin port).

```scala
get("/admin/finatra/users/",
  admin = true) { request: Request =>
  userDatabase.getAllUsers(
    request.params("cursor"))
}
```
<div></div>

Constant (no named parameters), HTTP method `GET` or `POST` routes can also be added to the [TwitterServer](http://twitter.github.io/twitter-server/) [HTTP Admin Interface](https://twitter.github.io/twitter-server/Admin.html) index.

To expose your route in the [TwitterServer](http://twitter.github.io/twitter-server/) [HTTP Admin Interface](https://twitter.github.io/twitter-server/Admin.html) index, the route path:

- **MUST** be a constant path
- **MUST** start with `/admin/`
- **MUST NOT** begin with `/admin/finatra/`
- **MUST** be HTTP method `GET` or `POST`

Set `admin = true` and optionally provide a [`RouteIndex`](https://github.com/twitter/finagle/blob/develop/finagle-http/src/main/scala/com/twitter/finagle/http/Route.scala), e.g.,

```scala
get("/admin/client_id.json",
  admin = true,
  index = Some(RouteIndex(alias = "Thrift Client Id", group = "Finatra")) ) { request: Request =>
  Map("client_id" -> "clientId.1234"))
}
```
<div></div>

The route will appear in the left-rail of the [TwitterServer](http://twitter.github.io/twitter-server/) [HTTP Admin Interface](https://twitter.github.io/twitter-server/Admin.html) under the heading specified by the `RouteIndex#group` indexed by `RouteIndex#alias` or the route's path. If you do not provide a `RouteIndex` the route will not appear in the index but is still reachable on the admin interface.

**Note**: all routes that start with *only* `/admin/` (and not `/admin/finatra/`) will be routed to by TwitterServer's [AdminHttpServer](https://github.com/twitter/twitter-server/blob/develop/src/main/scala/com/twitter/server/AdminHttpServer.scala#L108) and **not** by the Finatra [`c.t.finata.http.routing.HttpRouter`](https://github.com/twitter/finatra/blob/develop/http/src/main/scala/com/twitter/finatra/http/routing/HttpRouter.scala).

Thus any filter chain defined by your server's [`c.t.finata.http.routing.HttpRouter`](https://github.com/twitter/finatra/blob/develop/http/src/main/scala/com/twitter/finatra/http/routing/HttpRouter.scala) will **not** be applied to these routes. To maintain filtering as defined in the [`c.t.finata.http.routing.HttpRouter`](https://github.com/twitter/finatra/blob/develop/http/src/main/scala/com/twitter/finatra/http/routing/HttpRouter.scala) the routes **MUST** be under `/admin/finatra/` and are thus **not** eligible to be included in the [TwitterServer](http://twitter.github.io/twitter-server/) [HTTP Admin Interface](https://twitter.github.io/twitter-server/Admin.html) index.

## <a class="anchor" name="requests" href="#requests">Requests</a>
===============================

Each route has a callback which is executed when the route matches a request. Callbacks require explicit input types and Finatra will then try to convert the incoming request into the specified input type. Finatra supports two request types: a Finagle `http` Request or a custom `case class` request object.

### Finagle `c.t.finagle.http.Request`
This is a [c.t.finagle.http.Request](https://twitter.github.io/finagle/docs/index.html#com.twitter.finagle.http.Request) which contains common HTTP attributes.

### Custom `case class` request object
Custom request case classes are used for requests with a content-type of `application/json` and allow declarative request parsing with support for type conversions, default values, and validations.

For example, suppose you want to parse a `GET` request with three query params: `max`, `startDate`, and `verbose`, e.g.,

```text
http://foo.com/users?max=10&start_date=2014-05-30TZ&verbose=true
```
<div></div>

This can be modeled with the following `case class`:

```scala
case class UsersRequest(
  @Max(100) @QueryParam max: Int,
  @PastDate @QueryParam startDate: Option[DateTime],
  @QueryParam verbose: Boolean = false)
```
<div></div>

The custom `UsersRequest` `case class` can then be used as the route callback's input type:

```scala
get("/users") { request: UsersRequest =>
  request
}
```
<div></div>

##### Things to keep in mind:

- The `case class` field names should match the request parameters or use the [@JsonProperty](https://github.com/FasterXML/jackson-annotations#annotations-for-renaming-properties) annotation to specify the JSON field name in the case class (see: [example](https://github.com/twitter/finatra/blob/develop/jackson/src/test/scala/com/twitter/finatra/tests/json/internal/ExampleCaseClasses.scala#L141)). A [PropertyNamingStrategy](http://fasterxml.github.io/jackson-databind/javadoc/2.3.0/com/fasterxml/jackson/databind/PropertyNamingStrategy.html) can be configured to handle common name substitutions (e.g. snake_case or camelCase). By default, snake_case is used (defaults are set in [`FinatraJacksonModule`](https://github.com/twitter/finatra/tree/master/jackson/src/main/scala/com/twitter/finatra/json/modules/FinatraJacksonModule.scala)).
- Use a Scala ["back-quote" literal](http://www.scala-lang.org/files/archive/spec/2.11/01-lexical-syntax.html) for the field name when special characters are involved (e.g. ``@Header `user-agent`: String``).
- **Non-option fields without default values are considered required.** If a required field is missing, a `CaseClassMappingException` is thrown.
- The following field annotations specify where to parse a field out of the request
  * Request Fields:
     * [`@RouteParam`](https://github.com/twitter/finatra/blob/develop/http/src/test/scala/com/twitter/finatra/http/tests/integration/doeverything/main/domain/IdAndNameRequest.scala)
     * [`@QueryParam`](https://github.com/twitter/finatra/blob/develop/http/src/test/scala/com/twitter/finatra/http/tests/integration/doeverything/main/domain/RequestWithQueryParamSeqString.scala) - make sure that `@RouteParam` names do not collide with `@QueryParam` names. Otherwise, an `@QueryParam` could end up parsing an `@RouteParam`.
     * [`@FormParam`](https://github.com/twitter/finatra/blob/develop/http/src/test/scala/com/twitter/finatra/http/tests/integration/doeverything/main/domain/FormPostRequest.scala)
     * [`@Header`](https://github.com/twitter/finatra/blob/develop/http/src/test/scala/com/twitter/finatra/http/tests/integration/doeverything/main/domain/CreateUserRequest.scala)
     * [`@Inject`](https://github.com/twitter/finatra/blob/develop/http/src/test/scala/com/twitter/finatra/http/tests/integration/doeverything/main/domain/RequestWithInjections.scala) - can be used to inject any Guice managed class into your case class. It is not required for injecting the underlying Finagle `http` Request. To access the underlying Finagle `http` Request in your custom case class, simply include a field of type `c.t.finagle.http.Request`, for example:
```scala
case class CaseClassWithRequestField(
 @Header `user-agent`: String,
 @QueryParam verbose: Boolean = false,
 request: Request)
```
<div></div>

**<font color="red">Note:</font>** HTTP requests with a content-type of `application/json` sent to routes with a custom request case class callback input type will **always trigger** the parsing of the request body as well-formed JSON in attempt to convert the JSON into the request case class. This behavior can be disabled by annotating the case class with `@JsonIgnoreBody` leaving the raw request body accessible by simply adding a member of type `c.t.finagle.http.Request` as mentioned above.

For more specifics on how JSON parsing integrates with routing see: [Integration with Routing](/finatra/user-guide/json#routing-json) in the [JSON](/finatra/user-guide/json) documentation.

### <a class="anchor" name="forwarding-requests" href="#forwarding-requests">Request Forwarding</a>

You can forward a request to another controller. This is similar to other frameworks where forwarding will re-use the same request as opposed to issuing a redirect which will force a client to issue a new request.

To forward, you need to include a `c.t.finatra.http.request.HttpForward` instance in your controller, e.g.,

```scala
class MyController @Inject()(
  forward: HttpForward)
  extends Controller {
```
<div></div>

Then, to use in your route:

```scala
get("/foo") { request: Request =>
  forward(request, "/bar")
}
```
<div></div>

Forwarded requests will bypass the server defined filter chain (as the requests have already passed through the filter chain) but will still pass through controller defined filters.

For example, if a route is defined:

```scala
filter[MyAwesomeFilter].get("/bar") { request: Request =>
  "Hello, world."
}
```
<div></div>

When another controller forwards to this route, `MyAwesomeFilter` will be executed on the forwarded request.

### <a class="anchor" name="multipart-requests" href="#multipart-requests">Multipart Requests</a>

Finatra has support for multi-part requests. Here's an example of a multi-part `POST` controller route definition that simply returns all of the keys in the multi-part request:

```scala
post("/multipartParamsEcho") { request: Request =>
  RequestUtils.multiParams(request).keys
}
```
<div></div>

An example of testing this endpoint:

```scala
def deserializeRequest(name: String) = {
  val requestBytes = IOUtils.toByteArray(getClass.getResourceAsStream(name))
  Request.decodeBytes(requestBytes)
}

"post multipart" in {
  val request = deserializeRequest("/multipart/request-POST-android.bytes")
  request.uri = "/multipartParamsEcho"

  server.httpRequest(
    request = request,
    suppress = true,
    andExpect = Ok,
    withJsonBody = """["banner"]""")
}
```
<div></div>

For more information and examples, see:

- [`c.t.finatra.http.request.RequestUtils`](https://github.com/twitter/finatra/blob/develop/http/src/main/scala/com/twitter/finatra/http/request/RequestUtils.scala)
- [`c.t.finatra.http.fileupload.MultipartItem`](https://github.com/twitter/finatra/blob/develop/http/src/main/scala/com/twitter/finatra/http/fileupload/MultipartItem.scala)
- [`c.t.finagle.http.Request#decodeBytes`](https://github.com/twitter/finagle/blob/develop/finagle-http/src/main/scala/com/twitter/finagle/http/Request.scala#L192)
- [DoEverythingController](https://github.com/twitter/finatra/blob/develop/http/src/test/scala/com/twitter/finatra/http/tests/integration/doeverything/main/controllers/DoEverythingController.scala#L568)
- [DoEverythingServerFeatureTest](https://github.com/twitter/finatra/blob/develop/http/src/test/scala/com/twitter/finatra/http/tests/integration/doeverything/test/DoEverythingServerFeatureTest.scala#L332)
- [MultiParamsTest](https://github.com/twitter/finatra/blob/develop/http/src/test/scala/com/twitter/finatra/http/tests/request/MultiParamsTest.scala)

## <a class="anchor" name="responses" href="#responses">Responses</a>
===============================

### JSON responses

The simplest way to return a JSON response is to return a `case class` in your route callback. The default framework behavior is to render the `case class` as a JSON response E.g.,

```scala
case class ExampleCaseClass(
  id: String,
  description: String,
  longValue: Long,
  boolValue: Boolean)

get("/foo") { request: Request => 
  ExampleCaseClass(
    id = "123",
    description = "This is a JSON response body",
    longValue = 1L,
    boolValue = true)
}
```
<div></div>

will produce a response:

```
[Status]  Status(200)
[Header]  Content-Type -> application/json; charset=utf-8
[Header]  Server -> Finatra
[Header]  Date -> Wed, 17 Aug 2015 21:54:25 GMT
[Header]  Content-Length -> 90
{
  "id" : "123",
  "description" : "This is a JSON response body",
  "long_value" : 1,
  "bool_value" : true
}

```
<div></div>

**Note:** If you change the default [MessageBodyWriter](https://github.com/twitter/finatra/blob/develop/http/src/main/scala/com/twitter/finatra/http/internal/marshalling/FinatraDefaultMessageBodyWriter.scala) implementation (used by the [MessageBodyManager](https://github.com/twitter/finatra/blob/develop/http/src/main/scala/com/twitter/finatra/http/internal/marshalling/MessageBodyManager.scala)) this will no longer be the default behavior, depending. You can also always use the [ResponseBuilder](#response-builder) to explicitly render a JSON response.

### <a class="anchor" name="future-conversion" href="#future-conversion">Future Conversion</a>

For the basics of Futures in Finatra, see: [Futures](/finatra/user-guide/getting-started#futures) in the [Getting Started](/finatra/user-guide/getting-started) documentation.

Finatra will convert your route callbacks return type into a `c.t.util.Future[Response]` using the following rules:

* If you return a `c.t.util.Future[Response]`, then no conversion will be performed.
* `Some[T]` will be converted into a HTTP `200 OK`.
* `None` will be converted into a HTTP `404 NotFound`.
* Non-response classes will be converted into a HTTP `200 OK`.

Callbacks that do not return a [`c.t.util.Future`](https://github.com/twitter/util/blob/develop/util-core/src/main/scala/com/twitter/util/Future.scala) will have their return values wrapped in a [`c.t.util.ConstFuture`](https://twitter.github.io/util/docs/index.html#com.twitter.util.ConstFuture). If your non-future result calls a blocking method, you must [avoid blocking the Finagle request](https://twitter.github.io/scala_school/finagle.html#DontBlock) by wrapping your blocking operation in a FuturePool e.g.

```scala
import com.twitter.finatra.utils.FuturePools

class MyController extends Controller {

  private val futurePool = FuturePools.unboundedPool("CallbackConverter")

  get("/") { request: Request =>
    futurePool {
      blockingCall()
    }
  }
}
```
<div></div>

### <a class="anchor" name="response-builder" href="#response-builder">Response Builder</a>
All HTTP Controllers have a protected `response` field of type [`c.t.finatra.http.response.ResponseBuilder`](https://github.com/twitter/finatra/blob/develop/http/src/main/scala/com/twitter/finatra/http/response/ResponseBuilder.scala) which can be used to build callback responses. For example:

```scala
get("/foo") { request: Request =>
  ...
  
  response.
    ok.
    header("a", "b").
    json("""
    {
      "name": "Bob",
      "age": 19
    }
    """)
}

get("/foo") { request: Request =>
  ...

  response.
    status(999).
    body(bytes)
}

get("/redirect") { request: Request =>
  ...

  response
    .temporaryRedirect
    .location("/foo/123")
}

post("/users") { request: MyPostRequest =>
  ...

  response
    .created
    .location("/users/123")
}
```
<div></div>

For more examples, see the [ResponseBuilderTest](https://github.com/twitter/finatra/blob/develop/http/src/test/scala/com/twitter/finatra/http/tests/response/ResponseBuilderTest.scala).

### Cookies:
Cookies, like Headers, are read from request and can set via the [`c.t.finatra.http.response.ResponseBuilder`](https://github.com/twitter/finatra/blob/develop/http/src/main/scala/com/twitter/finatra/http/response/ResponseBuilder.scala#L151):

```scala
get("/") { request =>
  val loggedIn = request.cookies.getValue("loggedIn").getOrElse("false")
  response.ok.
    plain("logged in?:" + loggedIn)
}
```

```scala
get("/") { request =>
  response.ok.
    plain("hi").
    cookie("loggedIn", "true")
}
```
<div></div>

Advanced cookies are supported by creating and configuring [`c.t.finagle.http.Cookie`](https://github.com/twitter/finagle/blob/develop/finagle-http/src/main/scala/com/twitter/finagle/http/Cookie.scala) objects:

```scala
get("/") { request =>
  val c = new Cookie(name = "Biz", value = "Baz")
  c.setSecure(true)
  response.ok.
    plain("get:path").
    cookie(c)
}
```
<div></div>

### Response Exceptions:
Responses can be embedded inside exceptions with `.toException`. You can throw the exception to terminate control flow, or wrap it inside a `Future.exception` to return a failed `Future`. However, instead of directly returning error responses in this manner, a better convention is to handle application-specific exceptions in an [`ExceptionMapper`](/finatra/user-guide/build-new-http-server/exceptions.html).

```scala
get("/NotFound") { request: Request =>
  response.notFound("abc not found").toFutureException
}

get("/ServerError") { request: Request =>
  response.internalServerError.toFutureException
}

get("/ServiceUnavailable") { request: Request =>
  // can throw a raw exception too
  throw response.serviceUnavailable.toException
}
```
<div></div>

### Setting the Response Location Header:
ResponseBuilder has a "location" method.

```scala
post("/users") { request: Request =>
  response
    .created
    .location("/users/123")
}
```
<div></div>

which can be used:

   * if the URI starts with "http" or "/" then the URI is placed in the Location header unchanged.
   * `response.location("123")` will get turned into the correct full URL in the [HttpResponseFilter](https://github.com/twitter/finatra/blob/develop/http/src/main/scala/com/twitter/finatra/http/filters/HttpResponseFilter.scala) (e.g. `http://host.com/users/123`)

Or to obtain the request full path URL as follows:

```scala
RequestUtils.pathUrl(request)
```
<div></div>

Next section: [Add Filters](/finatra/user-guide/build-new-http-server/filter.html).

<nav>
  <ul class="pager">
    <li class="previous"><a href="/finatra/user-guide/build-new-http-server"><span aria-hidden="true">&larr;</span>&nbsp;Building&nbsp;a&nbsp;New&nbsp;HTTP&nbsp;Server</a></li>
    <li class="next"><a href="/finatra/user-guide/build-new-http-server/filter.html">Add&nbsp;Filters&nbsp;<span aria-hidden="true">&rarr;</span></a></li>
  </ul>
</nav>
