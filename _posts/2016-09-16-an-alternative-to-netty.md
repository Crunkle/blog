---
layout: post
title: An equally fast alternative to Netty
---

I was surprised to see a huge increase in views to this blog. I have taken a gap year and will be focusing on an upcoming project (or as the hippies call it: ‘exploring my inner being for a year’). This blog will serve as a public diary where I can express my thoughts and write posts on interesting, technological, findings.

In order to support thousands of concurrent clients at once, we need to avoid the use of blocking code in our networking. This is where the Java NIO package comes in to save the day. The only problem is, despite being a tad bit confusing to newcomers, it understandably doesn’t have many features. Netty is an event-driven networking framework which essentially allows you to pipe connections and parse socket data. It’s easy to use, fast and extremely versatile.

Why not just use Netty then? For starters, it (depending on the benchmark) isn’t always the quickest. A quick search result gave me Vert.x which is another event-driven asynchronous networking framework. Infact, benchmarks show that Vert.x was equally as fast, if not faster, than Netty itself. Interesting to say the least.

So how do you start a server in Vert.x? It is _extremely_ simple. I converted an entire project to use Vert.x recently, and was surprised at how quickly I could implement features.

```java
public class ExampleServer implements Runnable {

    public static void main(String... args) {
        new ExampleServer().run();
    }

    @Override
    public void run() {
        Vertx vertx = Vertx.vertx();

        NetServer netServer = vertx
                .createNetServer(
                        new NetServerOptions()
                                .setTcpKeepAlive(false)
                                .setIdleTimeout(30)
                );

        netServer.connectHandler(netSocket -> {
            netSocket.write("Here is a lovely stringy string for you.");
            netSocket.close();
        });

        netServer.listen(8080);
    }
}
```

Surprisingly enough, despite being powerful, that was it. I had a non-blocking server accepting connections and handling them in 40 lines of code. The only thing that threw me with this framework was the ‘verticles’. They could be compared to actors in an actor design pattern, which simplifies it a lot. You send data between verticles and change the state of the verticles in correlation with the data. This allows you to process it at different stages, similar to the Netty pipeline. You aren’t confined to this use and can easily treat them as individual threads receiving data.

![example](https://cdn.jared.im/static/firefox_2016-09-17_03-37-12.png)

Next time you whip out Netty, take some time to look for other alternatives. It is an amazing framework, and I am not saying that Vert.x is better, but it isn’t the only one out there. Just wanted to give a quick update to my blog whilst I find a better blog-like platform to host my site on.
