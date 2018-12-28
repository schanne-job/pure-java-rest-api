## REST API in pure Java without any frameworks

This is a demo application developed in Java 11 using 
[`jdk.httpserver`](https://docs.oracle.com/javase/10/docs/api/com/sun/net/httpserver/package-summary.html) module 
and a few additional Java libraries (like [vavr](http://www.vavr.io/), [lombok](https://projectlombok.org/)).

## Genesis of this project
I am a day-to-day Spring developer and I got used to this framework so much that I thought how it would be to forget about it for a while
and try to build such application from scratch. I thought it could be interesting from learning perspective and a bit refreshing.
When I started building this I often came across many situations when I missed some features which Spring provides out of the box.
This time, instead of turning on another Spring capability I had to rethink it and develop it myself.
It occurred that for real business case I would probably still prefer to use Spring instead of reinventing a wheel.
Still I believe the exercise was pretty interesting experience.

## Beginning.
I will go through this exercise step by step but not always pasting a complete code in text
but you can always checkout each step from a separate branch.
I started from empty `Application` main class. You can get an initial branch like that: 

```
git checkout step-1
```

## First endpoint

The starting point of the web application is `com.sun.net.httpserver.HttpServer` class. 
The most simple `/api/hello` endpoint could look as below: 

```java
import java.io.IOException;
import java.io.OutputStream;
import java.net.InetSocketAddress;

import com.sun.net.httpserver.HttpServer;

class Application {

    public static void main(String[] args) throws IOException {
        int serverPort = 8000;
        HttpServer server = HttpServer.create(new InetSocketAddress(serverPort), 0);
        server.createContext("/api/hello", (exchange -> {
            String respText = "Hello!";
            exchange.sendResponseHeaders(200, respText.getBytes().length);
            OutputStream output = exchange.getResponseBody();
            output.write(respText.getBytes());
            output.flush();
            exchange.close();
        }));
        server.setExecutor(null); // creates a default executor
        server.start();
    }
}
```
When you run main program it will start web server at port `8000` and expose out first endpoint which is just printing `Hello!`, e.g. using curl:

```bash
curl localhost:8000/api/hello
```

Try it out yourself from branch:

```bash
git checkout step-2
```

## Support different HTTP methods
Our first endpoint works like a charm but you will notice that no matter which HTTP method you'll use it will respond the same.
E.g.: 

```bash
curl -X POST localhost:8000/api/hello
curl -X PUT localhost:8000/api/hello
```

The first gotcha when building the API ourselves without a framework is that we need to add our own code to distinguish the methods, e.g.:

```java
        server.createContext("/api/hello", (exchange -> {

            if ("GET".equals(exchange.getRequestMethod())) {
                String respText = "Hello!";
                exchange.sendResponseHeaders(200, respText.getBytes().length);
                OutputStream output = exchange.getResponseBody();
                output.write(respText.getBytes());
                output.flush();
            } else {
                exchange.sendResponseHeaders(405, -1);// 405 Method Not Allowed
            }
            exchange.close();
        }));
```

Now try again request: 
```bash
curl -v -X POST localhost:8000/api/hello
```
and the response would be like: 

```bash
> POST /api/hello HTTP/1.1
> Host: localhost:8000
> User-Agent: curl/7.61.0
> Accept: */*
> 
< HTTP/1.1 405 Method Not Allowed
```

There are also a few things to remember, like to flush output or close exchange every time we return from the api.
When I used Spring I even did not have to think about it.

Try this part from branch:

```bash
git checkout step-3
```

## Parsing request params
Parsing request params is another "feature" which we'll need to implement ourselves in contrary to utilising a framework.
Let's say we would like our hello api to respond with a name passed as a param, e.g.: 

```bash
curl localhost:8000/api/hello?name=Marcin

Hello Marcin!

```
We could parse params with a method like: 

```java
public static Map<String, List<String>> splitQuery(String query) {
        if (query == null || "".equals(query)) {
            return Collections.emptyMap();
        }

        return Pattern.compile("&").splitAsStream(query)
            .map(s -> Arrays.copyOf(s.split("="), 2))
            .collect(groupingBy(s -> decode(s[0]), mapping(s -> decode(s[1]), toList())));

    }
```

and use it as below: 

```java
 Map<String, List<String>> params = splitQuery(exchange.getRequestURI().getRawQuery());
String noNameText = "Anonymous";
String name = params.getOrDefault("name", List.of(noNameText)).stream().findFirst().orElse(noNameText);
String respText = String.format("Hello %s!", name);
           
```

You can find complete example in branch:

```bash
git checkout step-4
```

Similarly if we wanted to use path params, e.g.: 

```bash
curl localhost:8000/api/items/1
```
to get item by id=1, we would need to parse the path ourselves to extract an id from it. This is getting cumbersome.


## Secure endpoint
A common case in each REST API is to protect some endpoints with credentials, e.g. using basic authentication.
For each server context we can set an authenticator as below: 

```java
HttpContext context =server.createContext("/api/hello", (exchange -> {
  // this part remains unchanged
}));
context.setAuthenticator(new BasicAuthenticator("myrealm") {
    @Override
    public boolean checkCredentials(String user, String pwd) {
        return user.equals("admin") && pwd.equals("admin");
    }
});
```

The "myrealm" in `BasicAuthenticator` is a realm name. Realm is a virtual name which can be used to separate different authentication spaces. 
You can read more about it in [RFC 1945](https://tools.ietf.org/html/rfc1945#section-11)

You can now invoke this protected endpoint by adding an `Authorization` header like that: 

```bash
curl -v localhost:8000/api/hello?name=Marcin -H 'Authorization: Basic YWRtaW46YWRtaW4='
```

The text after `Basic` is a Base64 encoded `admin:admin`  which are credentials hardcoded in our example code.
In real application to authenticate user you would probably get it from the header and compare with username and password store in database.
If you skip the header the API will respond with status
```
HTTP/1.1 401 Unauthorized

```

Check out the complete code from branch:

```bash
git checkout step-5
```