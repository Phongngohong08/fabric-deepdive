## Smart Contracts and Chaincode

Audience: Architects, App/Smart contract developers, Administrators

Tài liệu này tóm tắt smart contract và chaincode, kèm các trích đoạn API quan trọng và ví dụ.

### Khái niệm

- Smart contract: logic giao dịch tạo ra thay đổi trạng thái cho đối tượng nghiệp vụ; chạy trên peer.
- Chaincode: gói chứa một hoặc nhiều smart contract, được cài đặt/định nghĩa/commit trên kênh để sử dụng.

### Ledger APIs trong smart contract

Code minh hoạ (các API chính từ `ChaincodeStubInterface`):

```26:46:vendor/github.com/hyperledger/fabric-chaincode-go/v2/shim/interfaces.go
type ChaincodeStubInterface interface {
  GetArgs() [][]byte // tham số Invoke/Init
  GetFunctionAndParameters() (string, []string) // tên hàm + tham số
  GetTxID() string; GetChannelID() string
  InvokeChaincode(name string, args [][]byte, channel string) *peer.Response // gọi C2C
  GetState(key string) ([]byte, error) // đọc state
  PutState(key string, value []byte) error // ghi state (ghi vào writeset)
  DelState(key string) error // xoá key
  GetStateByRange(start, end string) (StateQueryIteratorInterface, error) // range query
  GetQueryResult(query string) (StateQueryIteratorInterface, error) // rich query (CouchDB)
  GetHistoryForKey(key string) (HistoryQueryIteratorInterface, error) // lịch sử key
  CreateCompositeKey(obj string, attrs []string) (string, error); SplitCompositeKey(k string) (string, []string, error)
  GetPrivateData(collection, key string) ([]byte, error); PutPrivateData(collection, key string, value []byte) error // private data
  SetEvent(name string, payload []byte) error // phát event
}
```

### Ví dụ tạo tài sản (Go)

```go
func (s *SmartContract) CreateAsset(ctx contractapi.TransactionContextInterface, id, color string, size int, owner string, appraised int) error {
  asset := Asset{ID: id, Color: color, Size: size, Owner: owner, AppraisedValue: appraised}
  b, _ := json.Marshal(asset)
  return ctx.GetStub().PutState(id, b)
}
```

### Endorsement policy (ở mức chaincode)

- Mọi smart contract trong cùng một chaincode dùng chung endorsement policy theo `chaincode definition`.
- Có thể dùng mặc định `Application/Endorsement` (ImplicitMeta) hoặc cung cấp Signature policy cụ thể khi approve.

### Giao dịch hợp lệ (tóm tắt)

1) Smart contract thực thi → trả về RW-set trong proposal response, chưa cập nhật world state.
2) Khi block được orderer phân phối, peer validate: đủ chữ ký theo policy và không vi phạm MVCC/phantom.
3) Chỉ giao dịch hợp lệ mới cập nhật world state; tất cả giao dịch (valid/invalid) đều ghi vào blockchain.

### Channels & lifecycle

- Cài đặt chaincode trên peer; các org approve định nghĩa (tên/phiên bản/policy/collections); khi đạt ngưỡng → commit trên kênh; khi đó toàn bộ smart contracts trong chaincode sẵn sàng sử dụng.

### System chaincode (tham khảo)

- `_lifecycle`: quản lý cài đặt/approve/commit chaincode.
- `CSCC`: cập nhật cấu hình kênh; `QSCC`: truy vấn ledger; `ESCC`/`VSCC`: ký phản hồi/validate.

---

Ghi chú:
- API trích dẫn đã được rút gọn cho nhanh tra cứu; xem file gốc để đầy đủ phương thức và chú thích MVCC/phantom.


