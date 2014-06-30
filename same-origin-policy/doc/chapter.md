# The Same Origin Policy

## Introduction

The same-origin policy (SOP) is part of the security mechanism of every modern browser. It controls when scripts running in a browser can communicate with one another (roughly, when they originate from the same website). First introduced in Netscape Navigator, the SOP now plays a critical role in the security of web applications; without it, it would be far easier for a malicious hacker to peruse your private photos on Facebook, or empty the balance on your bank account.

But the SOP is far from perfect. At times, it is too restrictive; there are cases (such as mashups) in which scripts from different origins should be able to share a resource but cannot. At other times, it is not restrictive enough, leaving corner cases that can be exploited using common attacks such as cross-site request forgery (CSRF). Furthermore, the design of the SOP has evolved organically over the years and puzzles many developers.

## Modeling with Alloy

This chapter is somewhat different from others in this book. Instead of building a working implementation, our goal is to construct an executable _model_ that serves as a simple yet precise description of the SOP. Like an implementation, the model can be executed to explore dynamic behaviors of the system; but unlike an implementation, the model omits low-level details that may get in the way of understanding the essential concepts.

To construct this model, we use _Alloy_, a language for modeling and analyzing software design. An Alloy model cannot be executed in the traditional sense of program execution. Instead, a model can be (1) _simulated_ to produce an _instance_, which represents a valid scenario or configuration of a system, and (2) _checked_ to see whether the model satisfies a desired _property_.

The approach we take might be called “agile modeling” because of its similarities to agile programming. We work incrementally, assembling the model bit by bit. Our evolving model is at every point something that can be executed. We formulate and run tests as we go, so that by the end we have not only the model itself but also a collection of properties that it satisfies. 

Despite these similarities, agile modeling differs from agile programming in one key respect. Although we'll be running tests, we actually won't be writing any. Alloy's analyzer generates test cases automatically, and all that needs to be provided is the property to be checked. Needless to say, this saves a lot of trouble (and text). The analyzer actually executes all possible test cases up to a certain size (called a _scope_); this typically means generating all starting states with at most some number of objects, and then choosing operations and arguments to apply up to some number of steps. Because so many tests are executed (typically billions), and because all possible configurations that a state can take are covered (albeit within the scope), this analysis tends to expose bugs more effectively than conventional testing.

## Simplifications

Because the SOP operates in the context of browsers, servers, the HTTP protocol, and so on, a complete description would be overwhelming. So our model (like all models) abstracts away irrelevant aspects, such as how network packets are structured and routed. But it also simplifies some relevant aspects, which means that the model cannot fully account for all possible security vulnerabilities.

For example, we treat HTTP requests like remote procedure calls, as if they occur at a single point in time, ignoring the fact that responses to requests might come out of order. We also assume that DNS (the domain name service) is static, so we cannot consider attacks in which a DNS binding changes during an interaction. In principle, though, it would be possible to extend our model to cover all these aspects, although it's in the very nature of security analysis that no model can expose all vulnerabilities, even if it represents the entire codebase.

## HTTP Protocol

The first step in building an Alloy model is to declare some sets of objects. Let's start with resources:

```
sig Resource {}
```

The keyword “sig” identifies this as an Alloy signature declaration. This introduces a set of resource objects; think of these, just like the objects of a class with no instance variables, as blobs that have identity but no contents. Resources are named by URLs (*uniform resource locators*):

```
sig Url {
  protocol: Protocol,
  host: Domain,
  port: lone Port,
  path: Path
}
sig Protocol, Domain, Port, Path {}
```
Here we have five signature declarations, introducing sets for URLs and each of the basic types of objects they comprise. Within the URL declaration, we have four fields. Fields are like instance variables in a class; if `u` is a URL, for example, then `u.protocol` would represent the protocol of that URL (just like dot in Java). But in fact, as we'll see later, these fields are relations. You can think of each one as if it were a two column database table. Thus `protocol` is a table with a column containing URLs and a column containing Protocols. And the innocuous looking dot operator is in fact a rather general kind of relational join, so that you could also write `protocol.p` for all the URLs with a protocol `p` -- but more on that later.

Note that domains and paths, unlike URLs, are treated as if they have no structure -- a simplification. The keyword `lone` (which can be read "less than or equal to one") says that each URL has at most one port. The path is the string that follows the host name in the URL, and which (for a simple static server) corresponds to the file path of the resource; we're assuming that it's always present, but can be an empty path.

Now we need some clients and servers:

```
abstract sig Endpoint {}
abstract sig Client extends Endpoint {}
abstract sig Server extends Endpoint {
  resources: Path -> lone Resource
}
```

The `extends` keyword introduces a subset, so the set `Client` of all clients, for example, is a subset of the set `Endpoint` of all endpoints. Extensions are disjoint, so no endpoint is both a client and a server. The `abstract` keyword says that all extensions of a signature exhaust it, so its occurrence in the declaration of `Endpoint`, for example, says that every endpoint must belong to one of the subsets (at this point, `Client` and `Server`). For a server `s`, the expression `s.resources` will denote a map from paths to resources (hence the arrow in the declaration). But as before, remember that each field is actually a relation that includes the owning signature as a first column, so this field represents a three-column relation on `Server`, `Path` and `Resource`.

This is a very simple model of a server: it has a static mapping of paths to resources. In general, the mapping is dynamic, but that won't matter for our analysis.

To map a URL to a server, we'll need to model DNS. So let's introduce a set `Dns` of domain name servers, each with a mapping from domains to servers:

```
one sig Dns {
  map: Domain -> Server
}
```

The keyword `one` means that (for simplicity) we're assuming exactly one domain name server; the expression `Dns.map` will represent a single, global mapping. The mapping is static -- another simplification. There are in fact known security attacks that rely on changing DNS bindings during an interaction, but we're ignoring that complication.

In order to model HTTP requests, we also need the concept of _cookies_, so let's declare them:

```
sig Cookie {
  domains: set Domain
}
```

Each cookie is scoped with a set of domains; this captures the fact that a cookie can apply to "*.mit.edu", which would include all domains with the suffix "mit.edu".

Finally, we can put this all together to construct a model of HTTP requests:

```
abstract sig HttpRequest extends Call {
  url: Url,
  sentCookies: set Cookie,
  body: lone Resource,
  receivedCookies: set Cookie,
  response: lone Resource,
}{
  from in Client
  to in Dns.map[url.host]
}
```

We're modeling an HTTP request and response in a single object; the `url`, `sentCookies` and `body` are sent by the client, and the `receivedCookies` and `response` are sent back by the server.

When writing the `HttpRequest` signature, we found that it contained generic features of calls, namely that they are from and to particular things. So we actually wrote a little Alloy module that declares the `Call` signature, and to use it here we need to import it:

```
open call[Endpoint]
```

It's a polymorphic module, so it's instantiated with `Endpoint`, the set of things calls are from and to.

Following the field declarations in `HttpRequest` is a collection of constraints. Each of these constraints applies to all members of the set of HTTP requests. The constraints say that (1) each request comes from a client, and (2) each request is sent to one of the servers specified by the URL host under the DNS mapping.

One of the prominent features of Alloy is that a model, no matter how simple or detailed, can be executed at any time to generate sample system instances. Let's use a `run` command to ask the Alloy Analyzer to execute the HTTP model that we have so far:

```
run {} for 3	-- generate an instance with up to 3 objects of every signature type
```

The analyzer conducts a systematic search for instances, and when it finds one that satisfies all the given constraints, it displays it graphically, like this:

![http-instance-1](fig-http-1.png)

This instance shows a client (represented by node `Client`) sending an `HttpRequest` to `Server`, which, in response, returns a resource object and instructs the client to store `Cookie` at `Domain`. 

Even though this instance is relatively trivial, it exposes an obvious flaw in our model. Although the server is holding a resource  (`Resource0`) that matches the path, it returned a different resources (`Resource1`) that does not actually exist on the server at all! Clearly, we neglected to specify an important constraint: that every response to a request must be a resource stored by the server against the given path. We can go back to our definition of `HttpRequest` and modify it accordingly:

```
abstract sig HttpRequest extends Call { ... }{
  ...
  response = to.resources[url.path]
}
```

Now if we run the analyzer again, we will see instances such as the following, in which the new constraint is taken into account:

![http-instance-1a](fig-http-1a.png)

Instead of generating sample instances, we can ask the analyzer to *check* whether the model satisfies a property. For example, one desirable property is that whenever a client sends the same request multiple times, it always receives the same response back:
```
check { all r1, r2: HttpRequest | r1.url = r2.url implies r1.response = r2.response } for 3 
```
Given this `check` command, the analyzer explores every possible behavior of the system (up to the specified bound), and as soon as it finds one that violates the property, it returns that instance as a *counterexample*:

![http-instance-2](fig-http-2.png)

This counterexample again shows an HTTP request being made by a client, but with two different servers (in Alloy, objects of the same type are distinguished by appending numeric suffixes to their names). Note that while the DNS server maps `Domain` to both `Server0` and `Server1` (in reality, this is a common practice for load balancing), only `Server0` maps `Path` to a resource object, causing `HttpRequest0` to result in an empty response! To fix this, we might add an Alloy *fact* saying that any two servers that DNS maps a single host to must provide the same set of resources:

```
fact ServerAssumption {
  all s1, s2 : Server | (some Dns.map.s1 & Dns.map.s2) implies s1.resources = s2.resources
}
```

When we re-run the `check` command after adding the fact, the analyzer no longer reports any counterexamples for the property.

## Browser

Let's introduce browsers:

```
sig Browser extends Client {
  documents: Document -> Time,
  cookies: Cookie -> Time,
}
```

This is our first example of a signature with "dynamic fields". Alloy has no built-in notions of time or behavior, which means that a variety of idioms can be used. In this model, we're using a common idiom in which you introduce a set of times `sig Time {}`
(a signature that is actually declared in the `call` module), and then you attach `Time` as a final column for every time-varying field. Take `cookies`, for example. As explained above (when we were talking about the `resources` field of `Server`), `cookies` is a relation with three columns. For a browser `b`, `b.cookies` will be a relation from cookies to time, and `b.cookies.t` will be the cookies held in `b` at time `t`. Likewise, the `documents` field associates a set of documents with each browser at a given time.

A document has a URL, some content and domain:

```
sig Document {
  src: Url,
  content: Resource -> Time,
  domain: Domain -> Time
}
```

The inclusion of the `Time` column for the last two tells us that they can vary over time, but the first (`src`, representing the source URL of the document) is fixed.

To model the effect of an HTTP request on a browser, we introduce a new signature, since not all HTTP requests will originate at the level of the browser; the rest will come from scripts.

```
sig BrowserHttpRequest extends HttpRequest {
  doc: Document
}{
  -- the request comes from a browser
  from in Browser
  -- the cookies that are sent were in the browser before the request
  sentCookies in from.cookies.before
  -- every sent cookie is scoped to the url of the request
  all c: sentCookies | url.host in c.domains
  -- a new document in the browser from which the request is sent
  documents.after = documents.before + from -> doc
  -- the new document has the response as its contents
  content.after = content.before ++ doc -> response
  -- the new document has the host of the url as its domain
  domain.after = domain.before ++ doc -> url.host
  -- the document's source field is the url of the request
  doc.src = url	
  -- the returned cookies are stored by the browser
  cookies.after = cookies.before + from -> sentCookies
}
```

This kind of request has one new field, `doc`, which is the document created in the browser from the resource returned by the request. As with `HttpRequest`, the behavior is described as a collection of constraints. Some of these say when the call can happen: for example, that the call has to come from a browser. Some of these constrain the arguments of the call: for example, that the cookies must be scoped appropriately. Some of these constrain the effect, and have a common form that relates the value of a relation after the call to its value before. For example, to understand

```
documents.after = documents.before + from -> doc
```

remember that `documents` is a 3-column relation on browsers, documents and times. The fields `before` and `after` come from the declaration of `Call` (which we haven't seen, but is included in the listing at the end), and represent the times before and after the call. The expression `documents.after` gives the mapping from browsers to documents after the call. So this constraint says that after the call, the mapping is the same, except for a new entry in the table mapping `from` to `doc`.

Some constraints use the `++` operator, which represents relational override (i.e, `e1 ++ e2` contains all tuples of `e2`, and additionally, any tuples of `e1` whose first element is not the first element of a tuple in `e2`). For example, the constraint

```
content.after = content.before ++ doc -> response
```

says that after the call, the `content` mapping is the mapping before, but updated to map `doc` to `response` (clobbering any previous mapping of `doc`). If we were to use `+` instead of `++`, the same document might map to multiple resources at the same time.

## Script

Next, we will build on the HTTP and browser models to introduce the notion of a *client-side script*, which represents a piece of code (typically in Javascript) executing inside a browser document (`context`). 
```
sig Script extends Client { context : Document }
```
A script is a dynamic entity that can perform two different types of actions: (1) it can make HTTP requests (i.e., Ajax requests) and (2) perform browser operations to manipulate the content and properties of a document. The flexibility of client-side scripts is one of the main catalysts behind the rapid development of Web 2.0, but it's also the reason why the SOP was created in the first place. Without the policy, scripts would be able to send arbitrary requests to servers, or freely modify the documents inside the browser -- which would be bad news if one or more of the scripts turned out to be malicious! 

A script can communicate to a server by sending an `XmlHttpRequest`:
```
sig XmlHttpRequest extends HttpRequest {}{
  from in Script
  noBrowserChange[before, after] and noDocumentChange[before, after]
}
```
An `XmlHttpRequest` can be used by a script to send/receive resources to/from a server, but unlike `BrowserHttpRequest`, it does not immediately result in creation of a new page or other changes to the browser and its documents. To say that a call does not modify the states of the system, we use predicates `noBrowserChange` and `noDocumentChange`:
```
pred noBrowserChange[before, after : Time] {
  documents.after = documents.before and cookies.after = cookies.before  
}
pred noDocumentChange[before, after : Time] {
  content.after = content.before and domain.after = domain.before  
}
```
What kind of actions can a script perform on documents? First, we introduce a generic notion of *browser operations* to represent a set of browser API functions that can be invoked by a script:
```
abstract sig BrowserOp extends Call { doc : Document }{
  from in Script and to in Browser
  doc + from.context in to.documents.before
  noBrowserChange[before, after]
}
```
Field `doc` refers to the document that will be accessed or manipulated by this call; additional parameters are inherited from `Call`, namely `from` and `to` representing the endpoints the browser operation is from and to, and `before` and `after` representing the times before and after the operation. The second constraint in the signature facts says that both `doc` and the document in which the script executes (`from.context`) must be documents that currently exist inside the browser. Finally, a `BrowserOp` may modify the state of a document, but not the set of documents or cookies* that are stored in the browser.

(* actually, cookies can be associated with a document and modified using a browser API, but we will omit this detail for now.)

A script can read from and write to various parts of a document, through a data structure called the "DOM" (document object model). The browser typically provides a collection of API functions for accessing the DOM. Their details are not relevant here, and it suffices to consider just two archetypal operations, `ReadDom` and `WriteDom`, that read and write the DOM respectively:
```
sig ReadDom extends BrowserOp { result : Resource }{
  result = doc.content.before
  noDocumentChange[before, after]
}
sig WriteDom extends BrowserOp { new_dom : Resource }{
  content.after = content.before ++ doc -> new_dom
  domain.after = domain.before
}
```
`ReadDom` returns the content of the target document, but does not modify it; `WriteDom`, on the other hand, sets the new content of the target document to `new_dom`. Note that the `doc` argument in these operations is unconstrained, so they are non-deterministic. When analyzing such a model, there is no need for the user to provide sample values; the analyzer will pick values in order to make constraints true (or to invalidate a property being checked).

In addition, a script can modify various properties of a document, such as its width, height, domain, and title. For the discussion of the SOP, we are only interested in the domain property, which can be modified by scripts using the `SetDomain` function:
```
sig SetDomain extends BrowserOp { new_domain : set Domain }{
  doc = from.context
  domain.after = domain.before ++ doc -> new_domain
  content.after = content.before
}
```
Why would you ever want to modify the domain property of a document? It turns out that this is one popular (but rather ad hoc) way of bypassing the SOP and allow cross-domain communication, which we will discuss in a later section.

Let's ask the Alloy Analyzer to generate instances with scripts in action:
```
run { some BrowserOp and some XmlHttpRequest} for 3 
```
One of the instances that it generates is as follows:

![script-instance-1](fig-script-1.png) 

In the first time step, `Script`, executing inside `Document0` from `Url1`, reads the content of another document from a different origin (`Url0`). Then, it sends the same content, `Resource1`, to `Server` by making an `XmlHtttpRequest` call. Imagine that `Document1` is your banking page, and `Document0` is an online forum injected with a malicious piece of code, `Script`. Clearly, this is not a desirable scenario, since your sensitive banking information is being relayed to a malicious server!

Another instance shows `Script` making an `XmlHttpRequest` to a server with a different domain:

![script-instance-2](fig-script-2.png)

Note that the request includes a cookie, which is scoped to the same domain as the destination server. This is potentially dangerous, because if the cookie is used to represent your identity (e.g., a session cookie), `Script` can effectively pretend to be you and trick the server into responding with your private data!

These two instances tell us that extra measures are needed to restrict the behavior of scripts, especially since some of those scripts could be malicious. This is exactly where the SOP comes in.

## Same Origin Policy

Before we can express the same origin policy, we must define what it means for two pages to have the *same* origin. Two URLs refer to the same origin if and only if they share the same hostname, protocol, and port:
```
pred sameOrigin[u1, u2 : Url] {
  u1.host = u2.host and u1.protocol = u2.protocol and u1.port = u2.port
}
```
The SOP itself has two parts, restricting the ability of a script to (1) make DOM API calls and (2) send HTTP requests. The first part of the policy states that a script can only read from and write to a document that comes from the same origin as the script:
```
pred domSop { all c: ReadDom + WriteDom | sameOrigin[c.doc.src, c.from.context.src] }
```
An instance such as the first script scenario is not possible under `domSop`, since `Script` is not allowed to invoke `ReadDom` on a document from a different origin.

The second part of the policy says that a script cannot send an HTTP request to a server unless its context has the same origin as the target URL -- effectively preventing instances such as the second script scenario.
```
pred xmlHttpReqSop { all x: XmlHttpRequest | sameOrigin[x.url, x.from.context.src] }
```
As we can see, the SOP is designed to prevent the two types of vulnerabilities that could arise from actions of a malicious script; without it, the web would be a much more dangerous place than it is today.

It turns out, however, that the SOP can be *too* restrictive. For example, sometimes you *do* want to allow communication between two documents of different origins. By the above definition of an origin, a script from `foo.example.com` would not be able to read the content of `bar.example.com`, or send a HTTP request to `www.example.com`, because these are all considered distinct hosts. 

In order to allow some form of cross-origin communication when necessary, browsers implemented a variety of mechanisms for relaxing the SOP. Some of these are more well thought out than others, and some have serious flaws that, when badly used, could negate the security benefits of the SOP. In the following sections, we will describe the most common of these mechanisms, and discuss their potential security pitfalls.

## Mechanisms for Bypassing the SOP

To be completed.
