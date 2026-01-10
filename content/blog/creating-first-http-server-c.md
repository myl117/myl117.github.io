+++
title = "Creating Your First HTTP Server in C"
date = "2026-01-10"
tags = [
    "c"
]
+++

If you're new to learning a low level programming language like C and have, like myself, come from the world of high level programming, building something as simple as an HTTP server can be quite daunting. But despite the perceived complexity, it is an awesome way to get more acquainted with the C syntax and essential skills like memory management, pointers and understanding core TCP/IP concepts.

## What is a TCP Socket?

TCP sockets are simply endpoints for communication between two machines over a network, using the Transmission Control Protocol (TCP). In practical terms, a socket is a combination of an IP address (we'll be sticking to IPV4), a protocol (TCP) and a port number. A socket acts like a two-way pipe. For two applications to communicate, they must both create a socket on their machines and then connect to each other to establish a communication channel.

HTTP (Hypertext Transfer Protocol) is just a text based protocol that is built on top of TCP so building a simple "Hello World" server like we are going to in this mini tutorial is fairly straightforward once we have a barebones TCP server set up.

## Creating a TCP Server

Before writing any code, we want to make sure we have added the necessary header files to the top of our entrypoint C file. Socket APIs are different depending on which operating system you use and since my daily driver is Windows, I will be using the Winsock API. If you are using a Unix/Linux based system, the code should be pretty similar but I will link a guide on setting up TCP sockets for Linux in the references section. In addition to using a socket API library, we will also need to add the header files for some standard libraries. Here is the full header import list:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <winsock2.h>
#include <ws2tcpip.h>
```

To begin using the Winsocks library and work with sockets, we need to initialise the networking subsystem.

```c
WSADATA wsa;

// This code turns on the networking subsystem so we can use sockets
if (WSAStartup(MAKEWORD(2,2), &wsa) != 0) {
  printf("Failed to initialize Winsock. Error Code: %d\n", WSAGetLastError());
  return 1;
}
```

Next, we want to create a TCP socket for our server. We also want to bind our newly created socket to an IP address and PORT and thereafter mark the socket as ready to accept connections. I've commented on the key parts of the code to help better explain what the various flags and variables are for.

```c
// create a socket
// AF_INET means IPV4 and SOCK_STREAM means TCP
server_sock = socket(AF_INET, SOCK_STREAM, 0);
if (server_sock == INVALID_SOCKET) {
  printf("Could not create socket. Error Code: %d\n", WSAGetLastError());
  WSACleanup();
  return 1;
}

// setup a data structure that describes an IPV4 and port
server_addr.sin_family = AF_INET; // IPV4
server_addr.sin_addr.s_addr = INADDR_ANY; // listen on all interfaces, localshost/wifi/ethernet
server_addr.sin_port = htons(3000); // port number
memset(server_addr.sin_zero, 0, sizeof(server_addr.sin_zero));

// we've previously created our socket but have not binded it to anything so it has no identity on the network
// so we need to bind it
if (bind(server_sock, (struct sockaddr *)&server_addr, sizeof(server_addr)) == SOCKET_ERROR) {
  printf("Bind failed. Error Code: %d\n", WSAGetLastError());
  closesocket(server_sock);
  WSACleanup();
  return 1;
}

// tell the OS to listen for incoming connections to our socket and accept TCP requests
// here we set the backlog of how many pending connections we can have on our socket to be 5
if (listen(server_sock, 5) == SOCKET_ERROR) {
  printf("Listen failed. Error Code: %d\n", WSAGetLastError());
  closesocket(server_sock);
  WSACleanup();
  return 1;
}
```

Now our socket is listening for connections but they aren't actually handled as of yet. To do that, we need to accept connections and also be able to handle new connections if they are made. To do that, we will use a combination of two while loops. The outer loop will accept new connections from clients and create a new client socket and the inner loop will continuously talk to the client until the client disconnects. Every time the client sends data, the server will echo the same data back.

```c
// outer loop - keep the server alive and accept connections
while(1) {
  client_sock = accept(server_sock, NULL, NULL);
  if (client_sock == INVALID_SOCKET) {
    printf("Accept failed. Error Code: %d\n", WSAGetLastError());
    closesocket(server_sock);
    WSACleanup();
    return 1;
  }

  printf("Client connected!\n");
  send(client_sock, "Welcome!\n", 9, 0);

  char buffer[1024];
  int bytes_read;

  // inner loop, keep talking to the client
  while ((bytes_read = recv(client_sock, buffer, sizeof(buffer) - 1, 0)) > 0) {
    buffer[bytes_read] = '\0'; // make it printable

    printf("Received: %s", buffer);
    send(client_sock, buffer, bytes_read, 0); // echo back
  }

  printf("Client disconnected\n");
  closesocket(client_sock);
}
```

And lastly, let's make sure that before we exit the application, we clean up Winsocks and close the server socket allowing our OS to free up memory.

```c
// Close server socket
closesocket(server_sock);

// Cleanup Winsock
WSACleanup();
```

To test the server, once built with your compiler of choice (I chose GCC) and running, we can use a tool like telnet or my personal favourite, ncat to make connections. You should see the server echo back any data you write in the terminal confirming the server is working.

## Creating an HTTP Server

An HTTP server, as mentioned earlier, is a protocol that is built on top of TCP so now that we have a server that can listen, accept, receive and send connections, the barebones for our HTTP server has already been completed.

The aim of our HTTP server is to send a "Hello World" message back to the client to be displayed in the browser. To do this, we want to only change the while loop that listens for connections and once a connection is made, construct a minimal HTTP response and send it to the client. An HTTP response consists of headers and a body where each header ends with "\r\n" and an empty line separates it from the body.

```c
// keep the server alive and accept HTTP connections
while(1) {
  client_sock = accept(server_sock, NULL, NULL);
  if (client_sock == INVALID_SOCKET) {
    printf("Accept failed. Error Code: %d\n", WSAGetLastError());
    closesocket(server_sock);
    WSACleanup();
    return 1;
  }

  printf("Client made a request!\n");

  char buffer[4096];

  int bytes_read = recv(client_sock, buffer, sizeof(buffer) - 1, 0);

  if (bytes_read <= 0) {
    closesocket(client_sock);
    continue;
  }

  buffer[bytes_read] = '\0';

  const char *http_response =
    "HTTP/1.1 200 OK\r\n"
    "Content-Type: text/plain\r\n"
    "Content-Length: 13\r\n"
    "\r\n"
    "Hello, world!";

  send(client_sock, http_response, strlen(http_response), 0);

  closesocket(client_sock);
}
```

Loading localhost:3000 in your browser should now display “Hello, world!”. And that’s it, we’ve accomplished our mission of creating a minimal HTTP server in C from scratch. It's a tad bit more technical than simply running a few commands and getting a Spring Boot or Express JS server up and running but the possibilities to expand this are endless. From here, you can implement more routes, support multiple file formats such as HTML and JSON or add multithreading to support multiple non-blocking client connections. I hope this mini tutorial will provide you with a launchpad for experimentation with networking in C.

Thanks for reading!

## References

[What is TCP/IP?](https://www.cloudflare.com/en-gb/learning/ddos/glossary/tcp-ip/)

[Ncat - Netcat for the 21st Century](https://nmap.org/ncat/)

[C/C++ -> Sockets Tutorial](https://www.linuxhowtos.org/C_C++/socket.htm)

[TCP Server-Client implementation in C](https://www.geeksforgeeks.org/c/tcp-server-client-implementation-in-c/)
