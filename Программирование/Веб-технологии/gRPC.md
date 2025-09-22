---
tags:
  - Программирование
  - ВебОсновы
  - RPC
  - gRPC
---
# gRPC — современный фреймворк RPC

**gRPC** — это современный фреймворк удалённого вызова процедур, разработанный Google. Он использует протокол HTTP/2 и Protocol Buffers для высокопроизводительной и компактной сериализации данных.

---

## Основные компоненты gRPC

- **Сервисы** определяются в `.proto` файлах.
- **Protocol Buffers (Protobuf)** — формат сериализации данных.
- **HTTP/2** — транспортный протокол с поддержкой мультиплексирования и сжатия заголовков.

---

## Пример определения сервиса (image_classifier.proto)

```protobuf
syntax = "proto3";

package imageclassifier;

service ImageClassifier {
  rpc Classify (ClassifyImageRequest) returns (ClassifyImageResponse);
}

message ClassifyImageRequest {
  bytes image_data = 1;
}

message ClassifyImageResponse {
  string label = 1;
  float confidence = 2;
}


---

## Генерация кода

```bash
protoc --go_out=. --go-grpc_out=. image_classifier.proto
```

---

## Пример сервера на Go

```go
package main

import (
  "context"
  "log"
  "net"

  "google.golang.org/grpc"
  pb "path/to/generated/proto"
)

type server struct {
  pb.UnimplementedImageClassifierServer
}

func (s *server) Classify(ctx context.Context, req *pb.ClassifyImageRequest) (*pb.ClassifyImageResponse, error) {
  // Простая заглушка для демонстрации
  return &pb.ClassifyImageResponse{
    Label:      "cat",
    Confidence: 0.98,
  }, nil
}

func main() {
  lis, err := net.Listen("tcp", ":50051")
  if err != nil {
    log.Fatalf("failed to listen: %v", err)
  }
  s := grpc.NewServer()
  pb.RegisterImageClassifierServer(s, &server{})
  if err := s.Serve(lis); err != nil {
    log.Fatalf("failed to serve: %v", err)
  }
}
```

---

## Пример клиента на Go

```go
package main

import (
  "context"
  "log"
  "time"

  "google.golang.org/grpc"
  pb "path/to/generated/proto"
)

func main() {
  conn, err := grpc.Dial("localhost:50051", grpc.WithInsecure(), grpc.WithBlock())
  if err != nil {
    log.Fatalf("did not connect: %v", err)
  }
  defer conn.Close()
  c := pb.NewImageClassifierClient(conn)

  ctx, cancel := context.WithTimeout(context.Background(), time.Second)
  defer cancel()

  resp, err := c.Classify(ctx, &pb.ClassifyImageRequest{
    ImageData: []byte{}, // Здесь должны быть данные изображения
  })
  if err != nil {
    log.Fatalf("could not classify: %v", err)
  }
  log.Printf("Prediction: %s, Confidence: %.2f", resp.Label, resp.Confidence)
}
```

---

## Преимущества gRPC

- Высокая производительность и компактность благодаря HTTP/2 и Protobuf  
- Поддержка множества языков программирования  
- Мультиплексирование и двунаправленные потоки данных  
- Строгая типизация и простота масштабирования сервисов  

---

Для детального изучения gRPC посетите официальный сайт: [https://grpc.io/](https://grpc.io/)

```