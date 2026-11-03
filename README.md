# Praktikum gRPC Polyglot

Server **Go** + Client **Python** + Client **Node.js** yang berkomunikasi lewat satu kontrak `user.proto`.

## Prasyarat

Pastikan tool berikut terpasang dan terdeteksi di terminal:

```bash
git --version
python --version
node -v
npm -v
go version
protoc --version
```

> Setelah `go install` (Langkah 3.2), pastikan folder `go\bin` (mis. `C:\Users\<nama>\go\bin`) ada di **PATH** agar `protoc-gen-go` dan `protoc-gen-go-grpc` ditemukan oleh `protoc`.

## Struktur Proyek

```
grpc-polyglot-lab/
├── .gitignore
├── user.proto
├── go-server/
│   ├── go.mod, go.sum
│   ├── main.go
│   └── pb/ (user.pb.go, user_grpc.pb.go)
├── python-client/
│   ├── client.py
│   ├── user_pb2.py
│   └── user_pb2_grpc.py
└── node-client/
    ├── client.js
    ├── package.json
    └── node_modules/
```

---

## 1. Setup Workspace dan Git

```bash
mkdir grpc-polyglot-lab
cd grpc-polyglot-lab
git init
```

Buat `.gitignore`:

```bash
echo "node_modules/" > .gitignore
echo "__pycache__/" >> .gitignore
echo ".venv/" >> .gitignore
echo "*.exe" >> .gitignore
```

Commit pertama:

```bash
git add .gitignore
git commit -m "Initial commit: Add gitignore rules"
```

## 2. Menulis Kontrak API (`user.proto`)

Buat `user.proto` di folder utama `grpc-polyglot-lab`:

```proto
syntax = "proto3";

package user;

option go_package = "./pb";

message UserRequest {
  int32 user_id = 1;
}

message UserResponse {
  string name = 1;
  string email = 2;
}

service UserService {
  rpc GetUserProfile (UserRequest) returns (UserResponse);
}
```

## 3. gRPC Server dengan Go

**3.1** Buat folder dan inisialisasi modul:

```bash
mkdir go-server
cd go-server
go mod init go-server
```

**3.2** Pasang dependensi dan plugin protoc:

```bash
go get google.golang.org/grpc
go get google.golang.org/protobuf
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest
```

**3.3** Generate kode Go (dari dalam folder `go-server`):

```bash
protoc --proto_path=../ --go_out=. --go-grpc_out=. ../user.proto
```

Hasil: folder `pb/` berisi `user.pb.go` dan `user_grpc.pb.go`.

**3.4** Buat `go-server/main.go`:

```go
package main

import (
	"context"
	"fmt"
	"log"
	"net"

	"go-server/pb"

	"google.golang.org/grpc"
)

type server struct {
	pb.UnimplementedUserServiceServer
}

func (s *server) GetUserProfile(ctx context.Context, req *pb.UserRequest) (*pb.UserResponse, error) {
	log.Printf("Menerima request untuk User ID: %d\n", req.GetUserId())
	if req.GetUserId() == 1 {
		return &pb.UserResponse{
			Name:  "Budi Santoso",
			Email: "budi@universitas.ac.id",
		}, nil
	}
	return &pb.UserResponse{
		Name:  "User Tidak Ditemukan",
		Email: "-",
	}, nil
}

func main() {
	lis, err := net.Listen("tcp", ":50051")
	if err != nil {
		log.Fatalf("Gagal listening port: %v", err)
	}

	s := grpc.NewServer()
	pb.RegisterUserServiceServer(s, &server{})

	fmt.Println("gRPC Server (Go) berjalan di port :50051...")
	if err := s.Serve(lis); err != nil {
		log.Fatalf("Gagal menjalankan server: %v", err)
	}
}
```

**3.5** Rapikan dependensi:

```bash
go mod tidy
```

## 4. gRPC Client dengan Python

**4.1** Kembali ke root, buat folder client:

```bash
cd ..
mkdir python-client
cd python-client
```

**4.2** Pasang library:

```bash
pip install grpcio grpcio-tools
```

**4.3** Generate kode Python:

```bash
python -m grpc_tools.protoc --proto_path=../ --python_out=. --grpc_python_out=. ../user.proto
```

Hasil: `user_pb2.py` dan `user_pb2_grpc.py`.

**4.4** Buat `python-client/client.py`:

```python
import grpc
import user_pb2
import user_pb2_grpc

def run():
    with grpc.insecure_channel('localhost:50051') as channel:
        stub = user_pb2_grpc.UserServiceStub(channel)
        print("Mengirim request ke Go Server menggunakan Python...")
        response = stub.GetUserProfile(user_pb2.UserRequest(user_id=1))
        print(f"Respon dari Server -> Nama: {response.name}, Email: {response.email}")

if __name__ == '__main__':
    run()
```

## 5. gRPC Client dengan Node.js

**5.1** Kembali ke root, buat folder dan inisialisasi npm:

```bash
cd ..
mkdir node-client
cd node-client
npm init -y
```

**5.2** Pasang library:

```bash
npm install @grpc/grpc-js @grpc/proto-loader
```

**5.3** Buat `node-client/client.js` (tanpa generate kode; proto dibaca saat runtime):

```javascript
const grpc = require('@grpc/grpc-js');
const protoLoader = require('@grpc/proto-loader');
const path = require('path');

const PROTO_PATH = path.join(__dirname, '../user.proto');
const packageDefinition = protoLoader.loadSync(PROTO_PATH, {
  keepCase: true,
  longs: String,
  enums: String,
  defaults: true,
  oneofs: true
});

const userProto = grpc.loadPackageDefinition(packageDefinition).user;

function main() {
  const client = new userProto.UserService('localhost:50051', grpc.credentials.createInsecure());
  console.log("Mengirim request ke Go Server menggunakan Node.js...");
  client.GetUserProfile({ user_id: 1 }, (error, response) => {
    if (!error) {
      console.log(`Respon dari Server -> Nama: ${response.name}, Email: ${response.email}`);
    } else {
      console.error("Terjadi error:", error);
    }
  });
}

main();
```

## 6. Pengujian

Buka **3 terminal terpisah**. Jalankan server Go lebih dulu.

**Terminal 1 – Server Go**

```bash
cd go-server
go run main.go
```

**Terminal 2 – Client Python**

```bash
cd python-client
python client.py
```

**Terminal 3 – Client Node.js**

```bash
cd node-client
node client.js
```

**Output yang diharapkan**

```
# Terminal 1
gRPC Server (Go) berjalan di port :50051...
2026/xx/xx xx:xx:xx Menerima request untuk User ID: 1
2026/xx/xx xx:xx:xx Menerima request untuk User ID: 1

# Terminal 2 / 3
Respon dari Server -> Nama: Budi Santoso, Email: budi@universitas.ac.id
```

**Kriteria keberhasilan**

- [ ] Tidak ada error `Connection refused` atau `Cannot find module` di ketiga terminal
- [ ] Data "Budi Santoso" tercetak di terminal Python dan Node.js
- [ ] Server Go mencetak log baru setiap ada client yang mengeksekusi kode

## 7. Push ke GitHub

Buat dulu repositori kosong di GitHub, lalu dari root proyek:

```bash
cd ..
git add .
git commit -m "Feat: Add full polyglot implementation of Go server, Python client, and Node client"
git branch -M main
git remote add origin https://github.com/USERNAME_ANDA/NAMA_REPO.git
git push -u origin main
```

---

## Tugas 1: Menambah Field `phone`

**1.** Ubah `user.proto`:

```proto
message UserResponse {
  string name = 1;
  string email = 2;
  string phone = 3;
}
```

**2.** Generate ulang kode Go:

```bash
cd go-server
protoc --proto_path=../ --go_out=. --go-grpc_out=. ../user.proto
```

**3.** Generate ulang kode Python:

```bash
cd ../python-client
python -m grpc_tools.protoc --proto_path=../ --python_out=. --grpc_python_out=. ../user.proto
```

**4.** Node.js tidak perlu generate ulang (proto dibaca saat runtime).

**5.** Ubah `go-server/main.go` agar mengisi `Phone`:

```go
if req.GetUserId() == 1 {
    return &pb.UserResponse{
        Name:  "Budi Santoso",
        Email: "budi@universitas.ac.id",
        Phone: "081234567890",
    }, nil
}
return &pb.UserResponse{
    Name:  "User Tidak Ditemukan",
    Email: "-",
    Phone: "-",
}, nil
```

**6.** Ubah `python-client/client.py`:

```python
print(f"Respon dari Server -> Nama: {response.name}, Email: {response.email}, Telepon: {response.phone}")
```

**7.** Ubah `node-client/client.js`:

```javascript
console.log(`Respon dari Server -> Nama: ${response.name}, Email: ${response.email}, Telepon: ${response.phone}`);
```

**8.** Restart server Go, lalu jalankan ulang kedua client:

```bash
# Terminal 1
cd go-server
go run main.go

# Terminal 2
cd python-client
python client.py

# Terminal 3
cd node-client
node client.js
```

**Output yang diharapkan**

```
Respon dari Server -> Nama: Budi Santoso, Email: budi@universitas.ac.id, Telepon: 081234567890
```

> **Catatan:** Server Go wajib di-restart setelah `main.go` dan kode hasil generate berubah, jika tidak field `phone` akan kosong.

---

## Troubleshooting

| Error | Solusi |
|---|---|
| `'go' is not recognized` | Instal Go dari go.dev, lalu buka terminal baru |
| `'protoc' is not recognized` | Ekstrak `protoc-win64.zip` ke `C:\protoc`, tambahkan `C:\protoc\bin` ke PATH, buka terminal baru |
| `protoc-gen-go: program not found` | Tambahkan `C:\Users\<nama>\go\bin` ke PATH, buka terminal baru |
| `Connection refused` / `UNAVAILABLE` | Jalankan server Go lebih dulu dan pastikan port `50051` sama |
| Field `phone` kosong | Generate ulang kode, lalu restart server Go |
