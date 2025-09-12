## Backend Communication design patterns

### Request Response pattern
Classic, simple, everywhere. A request is very generic so is response. 
**Steps involve:** Client send request -> Backend parse request (where req start, where it end) -> Process the request 
(understanding) -> Execute the request -> Response back -> Client parse and consume
**Usage:** Web, DNS, HTTP, SSH, RPC, SQL, REST, SOAP, GraphQL
**Chunking:** Multiple request & response can be sent until the end signal is sent, client can disconnect & comeback.
**Issues:** Server doesn't know anything about client until client hit a request. i.e.: For a notification service, how do 
we push the notification? There is no way but client pulling, which is the dirtiest way! Also, very long-running request 
is not good for RR. 

### Synchronous & Asynchronous pattern
By default, everything was synchronous, and async is modern & preferred. 
Caller & receiver are in sync in case of sync but not in async, sync blocks everything async don't, sync works in 
wait based async work on promise-future/callback/acknowledge base, sync works on single thread async works on multi 
thread & concurrency. Async is in everywhere: libraries, frameworks, databases, OS's, tool apps etc. 

### Push pattern
The client registers to a server and server pushes responses. It's a pre-request once - infinite response approach.
The server can send response any time it want, any amount it wants, as long as it wants. 
Good thing is its real time. But issues include: Client has to be online or miss data, client may not be able to 
handle load, requires bidirectional protocol. 

### Short polling
This is the technique people refer to with the term 'polling'. For a time taking intense process we may request the 
server to start working, server immediately responds with a handle, then eventually ask the server with handle 
"hey is it done?". Server would give response or say no, client keep asking accordingly. i.e.: ideal for file uploads. 
Good thing: client can now not get overwhelmed and get disconnected too unlike pushing. But things get dizzy for 
the server as it get too many requests from so many clients and most of them are useless! 

### Long polling
The ultimate solution to short polling's too many message problem: the main difference in long polling is, if the 
response is not ready the server don't return any response, goodbye to frequent "no" responses. Client only sends a new 
request after a "yes" response is received or a timeout occurs. As a result the client don't keep sending status requests. 
Once the response is ready it is returned. Good sides: its literally solves all issues. Bad sides: not very alive & 
practically realtime but not theoretically, waiting timeouts & queues - saves time tradeoffs. Kafka use long polling.

Two big questions: As the server holds the request so doesn't it cost extra space for that? 
And what about timeout? Is it only from client side or from server side too? 
Well there is no magic and the answers are negative as supposed to be! The server has timeouts after which it response
a "no" reply with 200 or 204 (No content), and the client has a longer timeout ideally, that acts after server. 
On the other hand, Yes, holding a request open for a long period does consume server resources. While it's more 
efficient than short polling, it's not without cost. Each open connection a server holds uses resources 
like memory and CPU. This is because the server needs to keep track of the client's connection state, often with a 
dedicated process or thread. For a small number of clients, this is negligible. However, in applications with thousands 
or millions of concurrent users (like a popular chat app), managing all these connections can become a significant 
challenge and a bottleneck for the server.

#### The resource consumption issue & Kafka
Kafka solves the resource consumption issue of long polling by employing a non-blocking I/O model. Instead of using a 
separate thread or process for each open connection, Kafka's brokers use a small number of threads to handle a large 
number of connections. This approach is highly efficient and scalable, allowing Kafka to manage many concurrent clients 
without a significant drain on server resources.
#### How It Works ⚙️
Kafka's solution is built on a fundamental shift from the traditional one-thread-per-connection model. 
Here's a breakdown of the key components:
- Non-Blocking I/O: Kafka's network layer is built on the Java NIO (New I/O) package. This allows a single thread to manage multiple socket connections. When a client sends a request, the broker doesn't block that thread while waiting for the response to be prepared. Instead, the thread is freed up to handle other connections. When the data is ready, the event is queued, and the thread can then asynchronously send the response back to the client.
- Single-Threaded Request Handler: A typical Kafka broker has a small number of network threads (I/O threads) that listen for incoming requests from clients. When a request arrives (e.g., a fetch request using long polling), it's placed in a request queue. A separate, single-threaded request handler processes these requests. This handler doesn't dedicate a thread to the client's connection. Instead, it processes the request and, if the data isn't immediately available (the core of the long polling wait), it puts the request on a "purgatory" queue.
- Purgatory Queue (Wait Queue): This is a crucial data structure for Kafka's long polling implementation. When a client's fetch request arrives and there's no new data available, Kafka doesn't hold the network thread. Instead, it places the request in the purgatory queue, where it "sleeps" without consuming any CPU cycles. Each request in the purgatory is associated with a timeout. As soon as new messages arrive for that topic-partition, Kafka awakens the relevant requests in the purgatory and moves them to a response queue to be processed and sent back to the client. Similarly, if the timeout expires, the request is also awakened and sent back to the client with an empty response.

By decoupling the network connection from the processing logic and using non-blocking I/O with a purgatory queue, Kafka effectively addresses the scalability challenges of long polling. It can maintain thousands of open, waiting connections without a significant resource penalty.

### Server sent events
It is just a single request and then a stream of unending responses from server. 
Pros & cons are same has push. So what's the difference? 
The main difference is that Server-Sent Events (SSE) are a unidirectional push model where the server sends updates to 
the client. Clients do send an initial request to the server to establish the connection, but after that, 
the communication is unidirectional. The client listens for events but can't send data back to the server in same connection. 
In a general push model, communication can be bidirectional, allowing both the server to push data to the client and 
the client to send data back to the server within the same connection.

### Pub-sub
What if a client want to talk to multiple server and a server want to serve multiple clients? A many-to-many (MESH) 
scenario. Pub-sub! There is set a queue which is the resource core, multiple servers access and writes in it, 
and multiple clients can read from it, a single end can act both as publisher and subscriber. These makes the model
loose-coupled, very dynamic, great for microservices, parallel processing, separated of concerns, scaling, no-problem if
offline but with a cost of complexity, network saturation & message delivery issues. 

### Sidecar pattern
The Sidecar Pattern is a microservices architecture design where a separate service, known as a sidecar, runs alongside 
the main application. This setup helps by offloading auxiliary tasks such as logging and monitoring from the main service, 
which allows it to focus on its core functionalities.
Benefits and Integration
Enhanced Maintainability and Scalability: By isolating auxiliary tasks, the sidecar pattern significantly boosts 
the maintainability and scalability of the application.
Technology Integration:
Kubernetes: Integrates seamlessly for orchestration, assisting in managing containerized applications.
Istio: Facilitates service meshes that help in promoting reusability and streamlined communication between services.
Considerations
Added Complexity: While beneficial, employing the sidecar pattern can introduce additional complexity.
Resource Usage: It may increase resource usage, so it's vital to monitor resource allocation.
Communication: Ensure clear communication channels between the main service and the sidecar to maintain efficient operation.