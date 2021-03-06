# gRPC-examples


gRPC examples and step by step guide of gRPC gateway. 

This repo also includes relevant links to get started with gRPC

## Official docs

https://grpc.io/docs

## PoC

### Goals

1. Create a protocol buffer file to define a greeter service
2. Implement a Java gRPC Server with that specification
3. Implement a Java gRPC Client with that specification
4. Implement a Gateway that handles both gRPC invocations and REST ones.

![Alt text](https://grpc.io/img/grpc-web-arch.png)

### Step by step

Import example project. Extracted from https://github.com/grpc/grpc-java/tree/master/examples. 

Based on steps from https://github.com/grpc/grpc-java/blob/master/examples/README.md


1. Create a protocol buffer file to define a greeter service

a. Created a hellorsk.proto that defines the Service interface, Messages and the Rest API endpoints.

b. Added the rskServer task to gradle configuration and `applicationDistribution`

c. `mvn compile`

d. `./gradlew installDist`

2. Implement a Java gRPC Server with that specification

The previous step will create the classes defined on the proto. Also, it will define the gRPC base service to implement. So the only thing remining is to implement it. 

The implementation is `RSKGreeterGrpc.RSKGreeterImplBase`

Those two classes were autogenerated by gradle tasks. 

Then, i have created a `HelloRskServer`, the gRPC server that exposes the gRPC service. 

To run it:

`./build/install/examples/bin/hello-rsk-server`

3. Implement a Java gRPC Client with that specification

Idem 2, you only need to implement the client. The `HelloRskClient` has a blocking stub to make gRPC calls to `HelloRskServer`

To run it: 

`./build/install/examples/bin/hello-rsk-client`

4. Implement a Gateway that handles both gRPC invocations and REST ones.

Based on https://grpc-ecosystem.github.io/grpc-gateway/docs/usage.html

For this step you need:

a. Install golang
b. Install protoc (the procol buff compiler). Download the latest release from https://github.com/protocolbuffers/protobuf/releases. For example https://github.com/protocolbuffers/protobuf/releases/download/v3.7.0/protoc-3.7.0-linux-x86_64.zip

With protoc we will create a gRPC stub, based on this anotation added to te proto file 

`import "google/api/annotations.proto";`

To generate a stub, run:

```
protoc -I/usr/local/include -I. \
  -I$GOPATH/src \
  -I$GOPATH/src/github.com/grpc-ecosystem/grpc-gateway/third_party/googleapis \
  --go_out=plugins=grpc:. \
  ./hellorsk.proto

```

This will generate a `hellorsk.pb.go` file. 

Then, as we have the client already running from step 2, we need a reverse proxy, we create a GO gateway with: 

```
  protoc -I/usr/local/include -I. \
  -I$GOPATH/src \
  -I$GOPATH/src/github.com/grpc-ecosystem/grpc-gateway/third_party/googleapis \
  --grpc-gateway_out=logtostderr=true:. \
  ./hellorsk.proto

```


Finally to create swagger docs: 

```
 protoc -I/usr/local/include -I. \
  -I$GOPATH/src \
  -I$GOPATH/src/github.com/grpc-ecosystem/grpc-gateway/third_party/googleapis \
  --swagger_out=logtostderr=true:. \
  ./hellorsk.proto
```

Now we need to create a GO entry file that runs up the Gateway: 

Basically, we need to define 

- where is the gateway (the gw imported folder)
- change the port to be used, this example uses 8080 to listen and 50051 as port to redirect gRPC traffic (on that port is the gRPC Server up and running)
- invoke the `RegisterRSKGreeterHandlerFromEndpoint` 

```
package main

import (
	"flag"
	"net/http"

	"github.com/golang/glog"
	"golang.org/x/net/context"
	"github.com/grpc-ecosystem/grpc-gateway/runtime"
	"google.golang.org/grpc"

	gw "./src/main/proto"
)

var (
	echoEndpoint = flag.String("hello_rsk", "localhost:50051", "Hello rsk endpoint")
)

func run() error {
	ctx := context.Background()
	ctx, cancel := context.WithCancel(ctx)
	defer cancel()

	mux := runtime.NewServeMux()
	opts := []grpc.DialOption{grpc.WithInsecure()}

	registerErr := gw.RegisterRSKGreeterHandlerFromEndpoint(ctx, mux, *echoEndpoint, opts)

	if registerErr != nil {
		return registerErr
	}


	return http.ListenAndServe(":8080", mux)
}

func main() {
	flag.Parse()
	defer glog.Flush()

	if err := run(); err != nil {
		glog.Fatal(err)
	}

}

```




Running them togheter: 

1. Run server 

```
marcos@marcos-rsk:~/grpc-java/examples$ ./build/install/examples/bin/hello-rsk-server
Mar 26, 2019 11:56:03 AM io.grpc.examples.hellorsk.HelloRskServer start
INFO: Server started, listening on 50051
```

2. 

Run the GO gateway

```
marcos@marcos-rsk:~/grpc-java/examples$ go run entry.go 

```

3.

REST: Test a POST request.

![Alt test](gateway.png?raw=true "REST")

GRPC: Test run the gRPC client 

![Alt test](gateway%202.png?raw=true "REST")





### Other interesting links

- Tutorials (NodeJS and Java tested): https://grpc.io/docs/tutorials/
- gRPC concepts: https://grpc.io/docs/guides/concepts.html
- Protocol buffers: https://developers.google.com/protocol-buffers/docs/overview 
- Java gateway project example https://github.com/wejrowski/grpc-gateway-java-gradle/blob/master/entry.go
- Web gRPC: https://github.com/grpc/grpc-web
- gRPC intro: https://www.baeldung.com/grpc-introduction
- Another tutorials: https://codelabs.developers.google.com/codelabs/cloud-grpc-java/index.html?index=..%2F..index#2

