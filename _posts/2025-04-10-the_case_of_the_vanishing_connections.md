---
layout: post
date: 2025-04-10
title: "The Case of the Vanishing Connections"
tags: go, debugging
---

# The Case of the Vanishing Connections

You know that sinking feeling when a test fails, but rerunning it magically makes it pass? That “uh oh, is this flaky?” moment. Yeah, we’ve been there. And recently, while working on the [go-github](https://github.com/google/go-github) client library, we hit one of those elusive intermittent failures that had us scratching our heads.
Here’s how we tracked it down, what we found, and how we fixed it.

# The Mystery
It started with this gem of a failure in GitHub Actions:

```
--- FAIL: TestUsersService_DeleteKey (0.00s)
    users_keys_test.go:170: Users.DeleteKey returned error: 
    Delete "http://127.0.0.1:40547/api-v3/user/keys/1": 
    net/http: HTTP/1.x transport connection broken: http: CloseIdleConnections called
```

No consistent pattern. No specific test always failing. Just a flaky mess that would go green when you hit "Re-run jobs". Classic.

# First Clue

The only thing that had changed? We’d recently added t.Parallel() to speed up the test suite. And while it did make things faster, it also introduced a new problem: flaky tests showing up like uninvited guests.  
That was our first real clue. This wasn’t a typical test failure. It had all the signs of a race condition. One that had been hiding quietly in the background until parallelism brought it into the spotlight.

# The Hidden Culprit: Shared Transport Pitfalls

Turns out the tests were unknowingly sharing the same underlying HTTP transport. Here's what was happening:

* Each test spins up its own HTTP server and client.
* But the clients don’t specify a custom transport.
* So they all fall back to Go’s http.DefaultTransport.

And this is where it gets spicy.  

When one test finishes, it calls server.Close(). This doesn’t just shut down the test server — it also calls CloseIdleConnections() on the shared default transport.  

So what happens if another test is running at that exact moment and tries to reuse a connection from that shared pool?  

Boom. “Connection broken.” 

# A Look Under the Hood

By default, if you don’t explicitly set a transport when creating an http.Client, Go will fall back to using the global http.DefaultTransport. You can see this in [Go’s source code](https://github.com/golang/go/blob/master/src/net/http/client.go#L60).  
So even if our tests were spinning up new clients and servers, they were still quietly sharing the same underlying transport.
And when a test server shuts down, it calls CloseIdleConnections() — on the default transport. You can see that in [httptest.Server.Close()](https://github.com/golang/go/blob/master/src/net/http/httptest/server.go#L237-L242)  
This is the key piece that causes the race condition. When one test finishes and shuts down its server, it closes idle connections across the shared transport — possibly interrupting other tests that are still running.

# Why It Was So Intermittent

This was a timing issue — a race condition in disguise:
1. Test A finishes and calls server.Close().
2. Go’s default transport clears idle connections, including ones Test B might want.
3. Test B tries to use a now-dead connection.
4. 💥 Test B fails.

But only sometimes, because this only happens if Test A and Test B hit that exact timing window. That’s what made it so infuriating to reproduce and debug.

# The Fix: Isolate Those Transports

Once we identified the shared http.Transport as the villain, the solution was simple: don’t share.  
Each test now gets its own http.Client with a unique Transport. That way, when one test closes its server and kills its connections, it doesn’t mess with others.  
Here’s the PR with the fix:  
👉 [#3529](https://github.com/google/go-github/pull/3529)  
And the diff is satisfyingly small. One of those “tiny change, huge impact” moments.  

# Lessons Learned

* Go’s http.DefaultTransport is global — respect it, or better yet, don’t use it in tests.
* Intermittent failures are usually timing bugs. If reruns help, look for shared state.
* Debugging tests is a great way to learn the real internals of the language.

# Final Thoughts

This was a fun (and slightly maddening) bug to track down, but it’s a good reminder: isolation in tests isn’t just about data — it’s also about infrastructure, like network connections.  
Next time your test randomly breaks with a weird “connection broken” message, maybe check who else is playing in the same sandbox.  
Happy debugging! 🐛🔍 
