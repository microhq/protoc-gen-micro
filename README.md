# protoc-gen-micro

This is protobuf code generation for micro. We use protoc-gen-micro to reduce boilerplate code.

## Install

```
go get github.com/micro/protoc-gen-micro
```

Also required: 

- [protoc](https://github.com/google/protobuf)
- [protoc-gen-go](https://github.com/golang/protobuf)

## Usage

Define your service as `greeter.proto`

```
syntax = "proto3";

service Greeter {
	rpc Hello(Request) returns (Response) {}
}

message Request {
	string name = 1;
}

message Response {
	string msg = 1;
}
```

Generate the code

```
protoc --proto_path=$GOPATH/src:. --micro_out=. --go_out=. greeter.proto
```

Your output result should be:

```
./
    greeter.proto	# original protobuf file
    greeter.pb.go	# auto-generated by protoc-gen-go
    greeter.micro.go	# auto-generated by protoc-gen-micro
```

The micro generated code includes clients and handlers which reduce boiler plate code

### Server

Register the handler with your micro server

```go
type Greeter struct{}

func (g *Greeter) Hello(ctx context.Context, req *proto.Request, rsp *proto.Response) error {
	rsp.Msg = "Hello " + req.Name
	return nil
}

proto.RegisterGreeterHandler(service.Server(), &Greeter{})
```

### Client

Create a service client with your micro client

```go
client := proto.NewGreeterService("greeter", service.Client())
```

### Errors

If you see an error about `protoc-gen-micro` not being found or executable, it's likely your environment may not be configured correctly. If you've already installed `protoc`, `protoc-gen-go`, and `protoc-gen-micro` ensure you've included `$GOPATH/bin` in your `PATH`.

Alternative specify the Go plugin paths as arguments to the `protoc` command

```
protoc --plugin=protoc-gen-go=$GOPATH/bin/protoc-gen-go --plugin=protoc-gen-micro=$GOPATH/bin/protoc-gen-micro --proto_path=$GOPATH/src:. --micro_out=. --go_out=. greeter.proto
```

### Endpoint

Add a micro API endpoint which routes directly to an RPC method

Usage:

1. Clone `github.com/googleapis/googleapis` to use this feature as it requires http annotations.
2. The protoc command must include `-I$GOPATH/src/github.com/googleapis/googleapis` for the annotations import.

```diff
syntax = "proto3";

import "google/api/annotations.proto";

service Greeter {
	rpc Hello(Request) returns (Response) {
		option (google.api.http) = { post: "/hello"; body: "*"; };
	}
}

message Request {
	string name = 1;
}

message Response {
	string msg = 1;
}
```

The proto generates a `RegisterGreeterHandler` function with a [api.Endpoint](https://godoc.org/github.com/micro/go-micro/api#Endpoint). 

```diff
func RegisterGreeterHandler(s server.Server, hdlr GreeterHandler, opts ...server.HandlerOption) error {
	type greeter interface {
		Hello(ctx context.Context, in *Request, out *Response) error
	}
	type Greeter struct {
		greeter
	}
	h := &greeterHandler{hdlr}
	opts = append(opts, api.WithEndpoint(&api.Endpoint{
		Name:    "Greeter.Hello",
		Path:    []string{"/hello"},
		Method:  []string{"POST"},
		Handler: "rpc",
	}))
	return s.Handle(s.NewHandler(&Greeter{h}, opts...))
}
```

## LICENSE

protoc-gen-micro is a liberal reuse of protoc-gen-go hence we maintain the original license 
