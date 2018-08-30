# Sync web clients data in realtime with NATS

REST API's are nice. Simple. Stateless. Scaleable. But to keep clients data updated in realtime, things get more complicated. Streaming events is easy, but you start having to deal with questions like:

<img align="right" alt="Wolf Questioning" src="wolf_questioning_150x240.png">

*Which resources do I need events for?*  
*How do I manage sending each client only the events they need?*  
*Can I prevent subscription to some events for unauthorized clients?*  
*Who keeps state of these event subscriptions?*  
*How do I handle lost connections? Or service crashes?*  
*Is it even possible to make search-, or pagination queries with realtime updates?*  
*Does it scale?*  
*Why can't it be made simpler?!*

Being the lead developer of the cloud offering at a leading provider of contact center solutions, I had to deal with these issues. And with the help of NATS I have a solution.

## Resgate - a Realtime API Gateway
The solution became the [*REsource Subscription (RES) Protocol*](https://github.com/jirenius/resgate/blob/master/docs/res-protocol.md), a simple JSON based protocol that revolves around the concept of *resources*, represented by JSON objects (*models*) and arrays (*collections*). And then [Resgate](https://github.com/jirenius/resgate), the gateway implementation that enables it all.

Resgate is a smart WebSocket-to-NATS (and REST-to-NATS) gateway, written in Go. It is similar to NATS in its high performance and simple setup. Apart from fetching resources, forwarding method calls, and passing on events, it also handles **access control**, **syncing**, **resource caching**, and more.

## NATS - the obvious choice
Resgate needed a messaging system, one that is **fast**, **reliable**, and **simple**, supporting both the *publish-subscribe* pattern as well as the *request-reply* pattern. NATS, with its admirable simplicity and performance, fitted the description like a glove.

The fact that NATS, just like Resgate, is written in Go, made the choice even easier. During the development of Resgate, NATS has also been used as a reference and inspiration.

## Fitting it all together
A simple NATS+Resgate setup would look like this:

<p align="center">
<img class="img-responsive center-block" alt="Architecture Diagram" src="simple-res-network-icon.svg">
</p>

The service(s) can be written in any language supported by NATS server. Instead of using HTTP, the service will listen to requests published over NATS. And if a resource is modified, the service will use NATS to publish the event that describes the modification.

> Rewrite this part. Instead of get/access, describe shortly the dual protocol setup.

Resgate becomes the single entry point for all clients. When a client requests a resource, Resgate will try to get it by publishing a *get* request over NATS (unless it is already in the cache). At the same time, it will also send an *access* request together with the client's access token, to verify authorization. Once both the *get*- and *access*-request returns, Resgate will send the resource (or an error) back to the client.

Does it sound complicated? It really isn't! Let me show you.

## Writing a service

Below are two javascript (node.js) snippets showing how to serve a resource (`models.mymodel`) using HTTP in comparison with Resgate:

**Using HTTP (with express):**
```js
var mymodel = { message: "Hello HTTP" };
// Listen to HTTP GET requests
app.get('/models/mymodel', function(req, resp) {
  resp.end(JSON.stringify(mymodel));
});
```

**Using Resgate (with nats):** 
```js
var mymodel = { message: "Hello NATS" };
// Listen to RES get requests over NATS
nats.subscribe('get.models.mymodel', function(req, reply) {
  nats.publish(reply, JSON.stringify({ result: { model: mymodel }}));
});

// Listen for access requests
nats.subscribe('access.models.mymodel', function(req, reply) {
  let { token } = JSON.parse(req);
  nats.publish(reply, JSON.stringify({ result: {
    get: true // Or false, if the token doesn't provide access
  }}));
});

// Updating the model
mymodel.message = "Hello NATS+Resgate";
nats.publish('event.models.mymodel.change', JSON.stringify({ message: mymodel.message  }));
```

<img align="right" alt="Wolf match maker" src="wolf_now_kiss_135x240.png">

That's it!

Let's take a look at the client side.

## Writing a client

There are two ways to get the resources from Resgate:

**Using HTTP:**

*GET: /api/models/mymodel*  
```js
{ "message": "Hello NATS" }
```

**Using javascript with ResClient:**
```js
let client = new ResClient('ws://api.example.com');
client.get('models.mymodel').then(model => {
    console.log(model.message); // Hello NATS
});
```

But when using ResClient, your resources are updated in realtime!
```js
let changeHandler = function() {
    console.log("Updated: " + model.message); // Updated: Hello NATS+Resgate
}

// Subscribe to events
model.on('change', changeHandler);

// Unsubscribe to events
model.off('change', changeHandler);
```

No extra code is needed to handle events that modifies the state. The resources are updated automatically by ResClient.

## Caching - taking load off the service

> Improve this one. Include concept of events.

Because the RES protocol has a concept of *resources*, it allows Resgate to cache resources requested by its clients. If multiple clients requests the same resource, it will only need to send a single *get* request to the service.

The cache is keep up-to-date using the events emitted by the service.

## ... and more
This blog post only deals with the basics on how to use NATS+Resgate to create realtime APIs for the web. There are more subjects to be introduced, such as:

* **resource queries** - for searches and pagination
* **method calls** - for calling methods (POST) on the resources
* **resource references** - for linking resources together
* **replication** - for scaling without limit
* **resynchronization** - for recovering from disconnects and crashes
* **hot-adding** - for adding/replacing services without disruption

But I'll leave that for another blog post.

## Now and beyond

<img align="right" style="margin: 8px 8px" alt="Wolf relaxing" src="wolf_relaxing_210x156.png">

 With NATS+Resgate and the REsource Subscription (RES) protocol, you can get realtime updates to your web clients while gaining functionality such as **end-user authentication**, **resource caching**, and **data-loss recovery**. And it is **high performant** and **simple**!

 While the project is young, the first version of the protocol is settled, where no changes will be added that breaks backwards compatability. Resgate will continue to get battle tested as the number of projects where the gateway is deployed in increases. Meanwhile, steps are being taken to provide a proper website with guides and examples to ease introduction and development of services for NATS+Resgate.
 
If you are interested in knowing more, visit the project page on [Github](https://github.com/jirenius/resgate).  
Or if you have any question or feedback, don't hesitate to contact me directly by e-mail:

[&#115;&#097;&#109;&#117;&#101;&#108;&#064;&#106;&#105;&#114;&#101;&#110;&#105;&#117;&#115;&#046;&#099;&#111;&#109;](mailto:&#115;&#097;&#109;&#117;&#101;&#108;&#064;&#106;&#105;&#114;&#101;&#110;&#105;&#117;&#115;&#046;&#099;&#111;&#109;)

Or find me in the [NATS Community](https://natsio.slack.com/messages/DBET737GV).

## Links
* **[Resgate](https://github.com/jirenius/resgate)** - project page for the realtime API gateway
* **[ResClient](https://www.npmjs.com/package/resclient)** - RES client library for javascript
* **[Resgate Test App](https://github.com/jirenius/resgate-test-app)** - test application used to test and develop Resgate
* **[go-res](https://github.com/jirenius/go-res)** - RES service library for Go

*NOTE: Resgate and all related tools are released under the MIT license.*
