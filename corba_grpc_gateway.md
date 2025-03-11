# C++ Orbix（CORBA）客戶端如何呼叫 Golang 伺服端？

## **問題：C++ Orbix 客戶端可以直接呼叫 Golang `net/rpc` 伺服端嗎？**
### **不可以，主要有以下原因：**
1. **通訊協議不同**
   - **C++ Orbix（CORBA）** 使用 **IIOP（Internet Inter-ORB Protocol）**。
   - **Golang `net/rpc`** 使用 **TCP-based RPC**，不相容 IIOP。

2. **序列化格式不同**
   - **CORBA 使用 CDR（Common Data Representation）**。
   - **Golang `net/rpc` 使用 GOB 編碼**，與 CORBA 不相容。

3. **服務發現方式不同**
   - **CORBA** 透過 **ORB（Object Request Broker）** 查找伺服端。
   - **Golang `net/rpc`** 透過 **TCP/IP 直接連接**。

---

## **推薦解法：C++ CORBA → gRPC Gateway**
為了讓 C++ Orbix 客戶端能夠與 Golang 伺服端通訊，建議：
- **C++ Orbix 客戶端 保持不變，仍然使用 CORBA 呼叫**
- **C++ CORBA 伺服端 轉發請求到 Golang gRPC 伺服端**

---

## **實作步驟**
### **步驟 1：C++ CORBA 伺服端 轉發請求至 Golang gRPC**
這裡的 C++ 伺服端負責：
1. **接受來自 C++ Orbix 客戶端的 IIOP 呼叫**。
2. **轉發請求到 Golang gRPC 伺服端**。

```cpp
#include <iostream>
#include "ExampleService.hh"
#include <grpcpp/grpcpp.h>
#include "example_service.grpc.pb.h"

class ExampleServiceImpl : public POA_Example::ExampleService {
public:
    char* GetMessage(const char* name) override {
        // 設置 gRPC 連接
        auto channel = grpc::CreateChannel("localhost:50051", grpc::InsecureChannelCredentials());
        std::unique_ptr<ExampleService::Stub> stub = ExampleService::NewStub(channel);

        // 構造 gRPC Request
        GetMessageRequest request;
        request.set_name(name);

        GetMessageResponse response;
        grpc::ClientContext context;

        // 發送 gRPC 請求
        grpc::Status status = stub->GetMessage(&context, request, &response);
        if (!status.ok()) {
            return CORBA::string_dup("gRPC Error");
        }

        return CORBA::string_dup(response.message().c_str());
    }
};
```

這樣，C++ ORB 伺服端會充當 **Gateway**，讓舊客戶端能夠繼續運作。

---

### **步驟 2：Golang gRPC 伺服端**
```go
package main

import (
	"context"
	"log"
	"net"

	pb "example.com/proto"
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

## **對應關係（C++ Orbix vs Golang gRPC Gateway）**
| CORBA Orbix (C++) | gRPC Gateway（C++ & Golang） |
|-------------------|-----------------------------|
| IDL (`.idl` 定義) | Protocol Buffers (`.proto`) |
| ORB（Object Request Broker） | gRPC 伺服端與 `ClientContext` |
| IIOP 傳輸協定 | HTTP/2（gRPC） |

---

## **結論**
- 因為 Golang 沒有 內建的 CORBA ORB，所以：
- **不能讓 C++ Orbix 直接呼叫 Golang `net/rpc` 伺服端**，因為通訊協議與序列化方式不相容。
- **最佳解法** 是 **讓 C++ CORBA 伺服端 充當 Gateway，轉發請求到 gRPC 伺服端**。
- 這樣可以 **讓舊的 C++ Orbix 客戶端 保持不變，但仍然能夠與 Golang 伺服端通訊**。

這樣就能讓 **舊系統逐步過渡到 Golang**！ 🚀
