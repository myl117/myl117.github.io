+++
title = "gRPC on the Frontend With gRPC-web"
date = "2026-01-14"
tags = [
  "grpc",
  "javascript",
  "typescript",
]
+++

We've previously explored gRPC and how it is a revolutionary protocol for microservice communication compared to the relatively archaic REST protocol. RPCs enable services to call procedures on other machines as if they were locally defined thanks to strongly typed Protobuf contracts. Protobufs also allow gRPC to encode data in binary form which is 5x more space and time efficient compared to JSON/XML. It may sound like I'm repeating myself (I promise I'm not sponsored by big-RPC) but gRPC really is the superior mode of communication for client-server communication.

So why is it not as widely used for browser and server communication? Simply put: browsers weren't built for gRPC and retrofitting them is awkward. gRPC requires support for features that browsers just don't have (or are limited) such as: HTTP/2, binary framing and streaming support among others.

This is where gRPC-web comes into play. Embracing the fact that browsers don't currently have the required feature set for traditional gRPC communication, it instead speaks a browser-safe protocol to a proxy (Envoy, nginx, etc.) which then converts it into real gRPC over HTTP/2. Here is a diagram that illustrates how this works:

![gRPC system architecture diagram](/images/grpc-frontend-grpc-web/diagram.jpg)
_gRPC system architecture diagram_

## Hands-on Implementation of gRPC-web

Okay, now that we are familiar with the fundamentals of how gRPC works, let's integrate it with the minimal library system we built in the previous blog post. Before we get started, I'd recommend enabling the proto reflection API on your gRPC server just in case you run into any configuration problems and then you can use grpcurl to confirm that the service is running and accessible. We'll be using Envoy as our translation proxy due to its simplicity and wide usage. Envoy is configured using a single yaml file. Make sure to replace the ports and server URIs with your own:

```yaml
static_resources:
  listeners:
    - name: listener_http
      address:
        socket_address:
          address: "0.0.0.0"
          port_value: 8282
      filter_chains:
        - filters:
            - name: envoy.filters.network.http_connection_manager
              typed_config:
                "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
                stat_prefix: ingress_http
                codec_type: AUTO
                route_config:
                  name: local_route
                  virtual_hosts:
                    - name: grpc_web_host
                      domains: ["*"]
                      cors:
                        allow_origin_string_match:
                          - exact: "http://localhost:8081"
                        allow_methods: "POST, GET, OPTIONS"
                        allow_headers: "content-type,x-grpc-web,x-user-agent,grpc-timeout"
                        expose_headers: "grpc-status,grpc-message"
                        max_age: "86400"
                      routes:
                        - match:
                            prefix: "/"
                          route:
                            cluster: grpc_backend
                            # disable streaming timeouts for gRPC
                            max_grpc_timeout: 0s
                http_filters:
                  - name: envoy.filters.http.cors
                    typed_config:
                      "@type": type.googleapis.com/envoy.extensions.filters.http.cors.v3.Cors
                  - name: envoy.filters.http.grpc_web
                    typed_config:
                      "@type": type.googleapis.com/envoy.extensions.filters.http.grpc_web.v3.GrpcWeb
                  - name: envoy.filters.http.router
                    typed_config:
                      "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router

  clusters:
    - name: grpc_backend
      dns_lookup_family: V4_ONLY
      connect_timeout: 5s
      type: LOGICAL_DNS
      lb_policy: ROUND_ROBIN
      # **HTTP/2 is required for gRPC backends**
      http2_protocol_options: {}
      load_assignment:
        cluster_name: grpc_backend
        endpoints:
          - lb_endpoints:
              - endpoint:
                  address:
                    socket_address:
                      address: host.docker.internal
                      port_value: 9090
```

Then using Docker, run Envoy with your yaml config file.

```bash
docker run --rm -p 8282:8282 -v "<absolute path to envoy folder>\envoy\envoy.yaml:/etc/envoy/envoy.yaml" envoyproxy/envoy:v1.36.0
```

Assuming your gRPC backend is running, your envoy server will now be ready to proxy requests. Now let's create a simple web application so that we can test requests from an actual browser. I'll be using webpack to bundle a static web application and run it.

```bash
mkdir grpc-library
cd grpc-library
npm init -y
npm install webpack webpack-cli webpack-dev-server html-webpack-plugin --save-dev
npm install google-protobuf grpc-web buffer
```

Next, let's generate proto files for CommonJS and gRPC-Web using the protoc compiler.

```bash
protoc ./proto/library.proto \
  --js_out=import_style=commonjs:./ \
  --grpc-web_out=import_style=commonjs,mode=grpcwebtext:./
```

In the webpack configuration, be sure to set the port number to the same one specified in your envoy.yaml. I'll be going with port 8081 as that is what I set in my Envoy config.

```js
const path = require("path");
const HtmlWebpackPlugin = require("html-webpack-plugin");

module.exports = {
  entry: "./index.js",
  output: {
    filename: "bundle.js",
    path: path.resolve(__dirname, "dist"),
  },
  mode: "development",
  devServer: {
    static: path.join(__dirname, "dist"),
    port: 8081,
  },
  plugins: [
    new HtmlWebpackPlugin({
      template: "./index.html",
    }),
  ],
  resolve: {
    fallback: {
      fs: false,
      path: false,
      buffer: require.resolve("buffer/"),
    },
  },
};
```

Our web server will contain an index.html and an index.js file. The latter will contain the core logic to call our gRPC methods defined in the generated proto stubs.

```js
import { LibraryServiceClient } from "./proto/library_grpc_web_pb";
import { BookRequest, Empty } from "./proto/library_pb";

// Connect to Envoy gRPC-Web proxy
const client = new LibraryServiceClient("http://localhost:8282", null, null);

// Get Book #1
const bookReq = new BookRequest();
bookReq.setId(1);

client.getBook(bookReq, {}, (err, res) => {
  if (err) {
    console.error("GetBook error:", err.message);
    return;
  }

  console.log("GetBook response:", {
    id: res.getId(),
    title: res.getTitle(),
    author: res.getAuthor(),
    available: res.getAvailable(),
  });
});

// List all books
const listReq = new Empty();

client.listBooks(listReq, {}, (err, res) => {
  if (err) {
    console.error("ListBooks error:", err.message);
    return;
  }

  const books = res.getBooksList().map((b) => ({
    id: b.getId(),
    title: b.getTitle(),
    author: b.getAuthor(),
    available: b.getAvailable(),
  }));

  console.log("ListBooks response:", books);
});
```

After running the webserver with webpack serve, you should see the response objects in the console. This sums up how efficient RPC protocols are. Developers can seamlessly call procedures locally without having to even consider that the data has travelled from a remote server, through a proxy and to the client. This small demo of gRPC-web showcases how powerful gRPC and Protobufs really are and how they are revolutionising the web development landscape.

Thanks for reading!

## References

[gRPC-web: Using gRPC in Your Front-End Application](https://torq.io/blog/grpc-web-using-grpc-in-your-front-end-application/)

[Lightning Talk: Connect from Browsers Using gRPC-Web - Stanley Cheung, Google](https://www.youtube.com/watch?v=PgnC4Pa89As)

[RPC vs REST - Difference Between API Architectures](https://aws.amazon.com/compare/the-difference-between-rpc-and-rest/)
