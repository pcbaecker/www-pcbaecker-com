---
title: "GRPC-Web example"
date: 2021-12-16T16:00:00Z
draft: false
---

There are dozens of articles out there describing the beneftits of GRPC over JSON. With [this](https://github.com/pcbaecker/grpc-web-example) little example I wanted to evaluate the complexity of GRPC in combination with a frontend. Because GRPC always uses HTTP/2.0 and that is not supported by all browsers there is GRPC-Web which translates between 'real' GRPC and and the web version for the browsers.

The project has three main components:
- The GRPC-Server
- The GRPC-Web proxy
- The Angular frontend

### Protobuf file

One benefit of GRPC is, that backend and frontend can generate their models and API's from the common .proto files. So I defined a basic book.proto file, which will be compiled into Golang and Typescript.

{{< highlight protobuf >}}
service BookService {
    rpc FindById(google.protobuf.UInt64Value) returns (Book) {}
    rpc FindAll(google.protobuf.Empty) returns (BookList) {}
    rpc Create(Book) returns (Book) {}
}

message Book {
    uint64 id = 1;
    string title = 2;
    string author = 3;
}
message BookList {
    repeated Book books = 1;
}
{{< / highlight >}}

[The .proto file](https://github.com/pcbaecker/grpc-web-example/blob/master/book.proto)

### Server

The server is just a mock that shows the basics of GRPC. It is a normal struct that must implement the service specific 'pb.UnimplementedBookServiceServer'. And can than override the functions defined in the .proto file like the following:

{{< highlight go >}}
type server struct {
	pb.UnimplementedBookServiceServer
}

func (s *server) FindById(ctx context.Context, id *wrapperspb.UInt64Value) (*pb.Book, error) {
	if len(books) < int(id.Value) {
		return &pb.Book{}, grpc.Errorf(codes.NotFound, "A book with id = "+strconv.Itoa(int(id.Value))+" could not be found!")
	}
	return books[id.Value], nil
}
{{< / highlight >}}

Starting the server is really straight forward. Special focus is on line 7, there we register our BookService into the generic GRPC-Server.

{{< highlight go "linenos=table" >}}
func main() {
	lis, err := net.Listen("tcp", fmt.Sprintf("localhost:%d", 9090))
	if err != nil {
		log.Fatalf("failed to listen: %v", err)
	}
	s := grpc.NewServer()
	pb.RegisterBookServiceServer(s, &server{})
	log.Printf("server listening at %v", lis.Addr())
	if err := s.Serve(lis); err != nil {
		log.Fatalf("failed to serve: %v", err)
	}
}
{{< / highlight >}}

[The servers main.go](https://github.com/pcbaecker/grpc-web-example/blob/master/grpc-server/main.go)

### GRPC-Web proxy

To translate the normal GRPC API's to the web we use the [GRPC-Web proxy](https://pkg.go.dev/github.com/improbable-eng/grpc-web/go/grpcwebproxy#section-readme) which is a lightweight way to achive this. For production use there may be the better alternative to do this with [Envoy Proxy](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/grpc_web_filter).
After installation the proxy can be started with a simple command:
{{< highlight bash >}}
./grpcwebproxy --backend_addr=localhost:9090 --run_tls_server=false --allow_all_origins
{{< / highlight >}}

### Frontend

With Angular and TypeScript we can easily use the generated TypeScript files to connect to the proxy and execute the requests. To create the client we need only one line:
{{< highlight typescript >}}
bookService = new BookServiceClient('http://localhost:8080', null, null);
{{< / highlight >}}

Loading all books from the server is also fairly easy:
{{< highlight typescript >}}
this.bookService.findAll(new Empty(), null).then((list) => {
      this.listOfBooks.data = list.getBooksList();
      console.log("Books sucessfully loaded");
}) 
{{< / highlight >}}

[Angular file](https://github.com/pcbaecker/grpc-web-example/blob/master/frontend/src/app/app.component.ts)

### Conclusion

In this article I just showed some key aspects of the project. The hardest thing to do was to generate the Go and TypeScript files from the .proto files. But even this was no deal breaker and took me just an hour to figure out. With that done the rest is pretty easy. What I really prefer over JSON is the single source of truth with the .proto files. It is much more readable in my opinion.

[Link to the project](https://github.com/pcbaecker/grpc-web-example)