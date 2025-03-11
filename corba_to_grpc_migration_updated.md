# C++ Orbix（CORBA）伺服端改為 Golang gRPC 的背景與改動方式

## 背景與原因
CORBA（Common Object Request Broker Architecture）是一種分散式物件技術，曾經廣泛用於企業級應用。然而，隨著時間的推移，CORBA 面臨多項挑戰，包括：
- **複雜度高**：IDL（Interface Definition Language）與 ORB（Object Request Broker）機制較為複雜。
- **可維護性差**：CORBA 技術過於陳舊，社群與廠商支援逐漸減少。
- **相容性問題**：不同 CORBA 實作之間的互通性不佳，尤其在跨語言或跨平台時易發生問題。
- **性能考量**：CORBA 的序列化與反序列化開銷較大，導致效能不如現代解決方案。

Golang gRPC 則是一種更現代、輕量且高效的 RPC 解決方案，具備：
- **效能優勢**：使用 Protocol Buffers 作為序列化格式，效能遠優於 CORBA。
- **簡單易用**：gRPC 提供清晰的 API 設計，且支援多語言。
- **跨平台與擴展性強**：基於 HTTP/2，可支援流式通訊與雙向傳輸。

因此，將原本的 C++ Orbix（CORBA）伺服端改為 **Golang gRPC**，能夠提升可維護性、效能以及與現代技術棧的相容性。

---

## 影響與客戶端改動方式
由於 gRPC 與 CORBA 的通訊協議、序列化格式、與傳輸層完全不同，**原本的 CORBA 客戶端也必須同步改造**，主要影響如下：

### 1. **通訊協議改變**
- **CORBA**：基於 IIOP（Internet Inter-ORB Protocol）。
- **gRPC**：基於 HTTP/2，使用 Protocol Buffers 進行序列化。

### 2. **IDL 轉換**
- CORBA 使用 IDL（.idl）來定義介面，gRPC 則使用 Protocol Buffers（.proto）。
- 必須將 CORBA IDL 轉換為 gRPC 的 `.proto` 檔案。

### 3. **客戶端改造**
- 移除 CORBA ORB 相關的初始化與呼叫。
- 改為使用 gRPC Stub 來呼叫 Golang 伺服端。

---

## 具體改造方式與範例

### **步驟 1：轉換 IDL 為 .proto**
假設原 CORBA IDL 為：
```idl
interface ExampleService {
  string GetMessage(in string name);
};
```
對應的 `.proto` 轉換為：
```proto
syntax = "proto3";

service ExampleService {
  rpc GetMessage (GetMessageRequest) returns (GetMessageResponse);
}

message GetMessageRequest {
  string name = 1;
}

message GetMessageResponse {
  string message = 1;
}
```

---

### **步驟 2：實作 Golang gRPC 伺服端**
```go
package main

import (
	"context"
	"log"
	"net"

	pb "example.com/proto" // 這裡指向編譯後的 gRPC package
	"google.golang.org/grpc"
)

type server struct {
	pb.UnimplementedExampleServiceServer
}

func (s *server) GetMessage(ctx context.Context, req *pb.GetMessageRequest) (*pb.GetMessageResponse, error) {
	return &pb.GetMessageResponse{Message: "Hello, " + req.Name}, nil
}

func main() {
	lis, err := net.Listen("tcp", ":50051")
	if err != nil {
		log.Fatalf("Failed to listen: %v", err)
	}

	grpcServer := grpc.NewServer()
	pb.RegisterExampleServiceServer(grpcServer, &server{})

	log.Println("Server is running on port 50051...")
	if err := grpcServer.Serve(lis); err != nil {
		log.Fatalf("Failed to serve: %v", err)
	}
}
```

---

### **步驟 3：改造 C++ 客戶端**
#### **原本的 CORBA 客戶端**
```cpp
#include <iostream>
#include "ExampleService.hh" // CORBA 產生的頭檔

int main(int argc, char* argv[]) {
    CORBA::ORB_var orb = CORBA::ORB_init(argc, argv);
    ExampleService_var service = ExampleService::_narrow(orb->resolve_initial_references("ExampleService"));
    
    CORBA::String_var response = service->GetMessage("Alice");
    std::cout << "Response: " << response.in() << std::endl;

    return 0;
}
```
#### **改為 gRPC C++ 客戶端**
```cpp
#include <iostream>
#include <grpcpp/grpcpp.h>
#include "example_service.grpc.pb.h"

int main() {
    auto channel = grpc::CreateChannel("localhost:50051", grpc::InsecureChannelCredentials());
    std::unique_ptr<ExampleService::Stub> stub = ExampleService::NewStub(channel);

    GetMessageRequest request;
    request.set_name("Alice");

    GetMessageResponse response;
    grpc::ClientContext context;

    grpc::Status status = stub->GetMessage(&context, request, &response);
    if (status.ok()) {
        std::cout << "Response: " << response.message() << std::endl;
    } else {
        std::cout << "gRPC Error: " << status.error_message() << std::endl;
    }

    return 0;
}
```

---

### **步驟 4：改為 gRPC Golang 客戶端**
```go
package main

import (
	"context"
	"fmt"
	"log"

	pb "example.com/proto" // 這裡指向編譯後的 gRPC package
	"google.golang.org/grpc"
)

func main() {
	conn, err := grpc.Dial("localhost:50051", grpc.WithInsecure())
	if err != nil {
		log.Fatalf("Failed to connect: %v", err)
	}
	defer conn.Close()

	client := pb.NewExampleServiceClient(conn)

	req := &pb.GetMessageRequest{Name: "Alice"}
	res, err := client.GetMessage(context.Background(), req)
	if err != nil {
		log.Fatalf("Error calling GetMessage: %v", err)
	}

	fmt.Println("Response:", res.Message)
}
```

---

## **結論**
將 **C++ Orbix（CORBA）伺服端改為 Golang gRPC**，不僅能夠提高效能與可維護性，還能簡化客戶端的開發流程。然而，因為底層通訊技術的變更，**客戶端也必須同步改造**，才能相容新的 gRPC 伺服端。

主要改造點：
1. **將 CORBA IDL 轉換為 `.proto`**。
2. **實作 Golang gRPC 伺服端**，提供相同的功能 API。
3. **改造 C++ 客戶端**，從 CORBA ORB 轉換為 gRPC Stub。
4. **提供 Golang gRPC 客戶端**，讓新客戶端能夠直接與 gRPC 伺服端溝通。

透過這種方式，能夠順利完成從舊技術（CORBA）到現代技術（gRPC）的轉移，提升系統的靈活性與長期可維護性。

