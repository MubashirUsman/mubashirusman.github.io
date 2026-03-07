---
layout: post
title:  "Concurrency Concepts And HTTP Server"
date:   2025-12-25 00:30:20 +0000
categories: programming python
---

## Concurrency
Its a way of fragmanting code so that individual fragmants can be run independently to reach the same result.  
E.g for taking an average of x1,x2,x3....xn, we can do it by dividing all numbers in two segments like s1 = sum(x1, x2, x3....xm)/c1 and s2 = sum(xm1, xm2...xn)/c2 and then doing something like (s1+s2)/(c1+c2). This can be done by running fragmants on different cpu cores at the same time (parallel execution) or by sharing the same cpu (time-sliced execution).  

## HTTP Server
An HTTP server does the following:  
1. Create a TCP/steam socket
2. Bind a name (address) to this socket object
3. Start listening, which is to wait for incoming connections
4. When a connection comes, accept the connection and start sharing HTTP messages
5. Close the connection
6. Repeat from step 3

Before writing any code, we need to know what a socket is: its a channel for communication for intra-computer communication, there are _client_ sockets and _server_ sockets. Client socket sends a text, receives a reply. After it exchanges some messages client socket is then destroyed.  
```
seversocket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
seversocket.bind(('127.0.0.1', 8765))
serversocket.listen(5) #max number of connect requests
```
Now we can enter the mainloop for accepting connections, we can get the client socket from the `accept` and fulfill the request.
