## Private data

Audience: Architects, Application/Smart contract developers, Administrators

Tài liệu này tóm tắt Private Data Collections (PDC) và chèn trích đoạn mã/đường dẫn hữu ích.

### Khái niệm

- PDC cho phép một tập con tổ chức trong cùng kênh lưu trữ dữ liệu bí mật (private state) trên peers của họ, trong khi toàn bộ kênh chỉ thấy hash.
- Private data đi qua gossip P2P giữa các org được uỷ quyền; hash được order, đưa vào block, và dùng để validate/audit.

### API chaincode đối với private data

```259:299:vendor/github.com/hyperledger/fabric-chaincode-go/v2/shim/interfaces.go
GetPrivateData(collection, key string) ([]byte, error)
PutPrivateData(collection, key string, value []byte) error
DelPrivateData(collection, key string) error
PurgePrivateData(collection, key string) error // xoá hoàn toàn (v2.3+)
GetPrivateDataHash(collection, key string) ([]byte, error) // đọc hash từ non‑member peer
```

### Implicit per‑org collections

- Mỗi chaincode có các implicit collections theo mẫu `_implicit_org_<MSPID>`, không cần định nghĩa trước; policy và dissemination mặc định theo chính org đó.

Code minh hoạ (tạo tên collection ngầm định):

```17:20:core/chaincode/implicitcollection/name.go
func NameForOrg(mspid string) string { return "_implicit_org_" + mspid }
```

Code minh hoạ (tạo cấu hình implicit collection cho từng org trên kênh):

```243:262:core/chaincode/lifecycle/deployedcc_infoprovider.go
func (vc *ValidatorCommitter) GenerateImplicitCollectionForOrg(mspid string) *pb.StaticCollectionConfig {
  requiredPeerCount, maxPeerCount := 0, 0
  if mspid == vc.CoreConfig.LocalMSPID { // với org của peer hiện tại có thể lấy giá trị từ core.yaml
    requiredPeerCount = vc.PrivdataConfig.ImplicitCollDisseminationPolicy.RequiredPeerCount
    maxPeerCount = vc.PrivdataConfig.ImplicitCollDisseminationPolicy.MaxPeerCount
  }
  return &pb.StaticCollectionConfig{
    Name: implicitcollection.NameForOrg(mspid),
    MemberOrgsPolicy: &pb.CollectionPolicyConfig{ Payload: &pb.CollectionPolicyConfig_SignaturePolicy{ SignaturePolicy: policydsl.SignedByMspMember(mspid) }},
    RequiredPeerCount: int32(requiredPeerCount),
    MaximumPeerCount:  int32(maxPeerCount),
  }
}
```

Sử dụng trong chaincode (ghi/đọc implicit collection):

```go
coll := "_implicit_org_Org1MSP"
_ = ctx.GetStub().PutPrivateData(coll, keyHash, valueBytes)
value, _ := ctx.GetStub().GetPrivateData(coll, keyHash)
```

### Luồng giao dịch có private data (tóm tắt)

1) Client gửi proposal kèm dữ liệu nhạy cảm qua `transient` tới peer mục tiêu; Gateway chọn các endorsing peers thuộc các org thành viên PDC.
2) Endorsing peers chạy chaincode, lưu private data vào transient store cục bộ, phát tán P2P tới orgs được phép; proposal response chỉ chứa hash private data.
3) Transaction (chứa hash) được orderer cắt khối và phân phối như bình thường.
4) Khi commit, peer có quyền sẽ kéo private data (nếu chưa có), so khớp hash, sau đó ghi vào private state DB và private writeset storage, đồng thời xoá khỏi transient store.

### Chia sẻ/kiểm chứng private data

- Non‑member peers/chaincode có thể dùng `GetPrivateDataHash()` để kiểm chứng dữ liệu nhận off‑chain khớp hash on‑chain.
- Có thể copy/transfer giữa collections bằng cách dùng transient + kiểm tra hash trước khi ghi.

### Purge private data

- Dùng `PurgePrivateData(collection, key)` để gỡ hoàn toàn dữ liệu khỏi các peers thành viên; khác với `DelPrivateData` (chỉ xoá state hiện tại nhưng còn lịch sử để sync/emit events).
- PDC có thể cấu hình auto‑purge sau N block không sửa đổi.

---

Ghi chú:
- Cần cấu hình anchor peers và `CORE_PEER_GOSSIP_EXTERNALENDPOINT` để gossip xuyên tổ chức hoạt động khi dùng PDC.


