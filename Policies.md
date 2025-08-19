## Policies

Audience: Architects, Application/Smart contract developers, Administrators

Tài liệu này tóm tắt chính sách trong Fabric và chèn các trích đoạn mã quan trọng: định nghĩa, kiểu, cách viết và sử dụng trong cấu hình/kênh/chaincode lifecycle.

### What is a policy & Why needed

- Policy là tập quy tắc xác định ai (principals/organizations) được phép làm gì và cần bao nhiêu chữ ký để thông qua một hành động (thay đổi cấu hình, endorse chaincode, nhận block events, v.v.).
- Trong Fabric, mọi thao tác quan trọng đều chịu sự kiểm soát của policy và có thể thay đổi theo thời gian thông qua cập nhật cấu hình kênh.

### How are policies implemented

- Chính sách được lưu trong cấu hình kênh theo cây `Channel` → `Application`/`Orderer` → `<Org>` và được mô tả bởi thông điệp `common.Policy`.

Code minh hoạ (kiểu `Policy` và `SignaturePolicyEnvelope`):

```28:37:vendor/github.com/hyperledger/fabric-protos-go-apiv2/common/policies.pb.go
type Policy_PolicyType int32
const (
    Policy_UNKNOWN = 0
    Policy_SIGNATURE = 1
    Policy_MSP = 2
    Policy_IMPLICIT_META = 3
)
```

```130:138:vendor/github.com/hyperledger/fabric-protos-go-apiv2/common/policies.pb.go
type Policy struct {
    Type  int32  // Kiểu policy (SIGNATURE/IMPLICIT_META)
    Value []byte // Nội dung policy đã marshal (SignaturePolicyEnvelope/ImplicitMeta)
}
```

```186:195:vendor/github.com/hyperledger/fabric-protos-go-apiv2/common/policies.pb.go
type SignaturePolicyEnvelope struct {
    Version int32
    Rule *SignaturePolicy            // Luật NOutOf/ SignedBy
    Identities []*msp.MSPPrincipal   // Danh sách principals dùng trong Rule
}
```

### Signature vs ImplicitMeta

- **Signature**: viết trực tiếp các yêu cầu chữ ký theo principals (ví dụ OR/AND/NOutOf các vai trò `'Org1MSP.peer'`, `'Org2MSP.admin'`...).
- **ImplicitMeta**: chỉ dùng trong cấu hình kênh; tổng hợp kết quả của các sub‑policies bên dưới theo luật `ANY|ALL|MAJORITY <SubPolicyName>`.

Code minh hoạ (tạo `ImplicitMeta` bằng util):

```39:47:common/policies/util.go
func makeImplicitMetaPolicy(subPolicyName string, rule cb.ImplicitMetaPolicy_Rule) *cb.Policy {
    return &cb.Policy{ Type: int32(cb.Policy_IMPLICIT_META), Value: protoutil.MarshalOrPanic(&cb.ImplicitMetaPolicy{ Rule: rule, SubPolicy: subPolicyName }) }
}
```

Code minh hoạ (DSL tạo `Signature` policy theo vai trò MSP):

```61:76:common/policydsl/policydsl_builder.go
func SignedByMspPeer(mspId string) *cb.SignaturePolicyEnvelope { // 1 chữ ký từ bất kỳ peer của MSP
    return signedByFabricEntity(mspId, mb.MSPRole_PEER)
}
``;

### Viết policy trong configtx.yaml

- Ở cấp tổ chức, các policy `Readers/Writers/Admins/Endorsement` thường dùng `Signature`:

```214:239:policies-raw.txt
Policies:
  Readers:     { Type: Signature, Rule: "OR('Org1MSP.admin','Org1MSP.peer','Org1MSP.client')" }
  Writers:     { Type: Signature, Rule: "OR('Org1MSP.admin','Org1MSP.client')" }
  Admins:      { Type: Signature, Rule: "OR('Org1MSP.admin')" }
  Endorsement: { Type: Signature, Rule: "OR('Org1MSP.peer')" }
```

- Ở cấp `Application`, các policy tổng hợp dùng `ImplicitMeta`:

```284:299:policies-raw.txt
Policies:
  Readers:             { Type: ImplicitMeta, Rule: "ANY Readers" }
  Writers:             { Type: ImplicitMeta, Rule: "ANY Writers" }
  Admins:              { Type: ImplicitMeta, Rule: "MAJORITY Admins" }
  LifecycleEndorsement:{ Type: ImplicitMeta, Rule: "MAJORITY Endorsement" }
  Endorsement:         { Type: ImplicitMeta, Rule: "MAJORITY Endorsement" }
```

### Fabric chaincode lifecycle và chính sách

- Khi approve/commit chaincode, hai policy thường gặp:
  - `Application/LifecycleEndorsement`: ai phải approve định nghĩa chaincode.
  - `Application/Endorsement`: mặc định policy endorse chaincode (có thể thay bằng Signature policy cụ thể).

### ACLs

- ACL ánh xạ tài nguyên nội bộ (ví dụ `peer/ChaincodeToChaincode`) tới một policy đường dẫn `/Channel/Application/Writers` để kiểm soát ai được phép gọi.

### Một số hằng số đường dẫn chuẩn

```23:51:common/policies/policy.go
const (
  PathSeparator = "/"
  ChannelPrefix = "Channel"
  ApplicationPrefix = "Application"
  OrdererPrefix = "Orderer"
  ChannelReaders = "/Channel/Readers"
  ChannelWriters = "/Channel/Writers"
  ChannelApplicationAdmins = "/Channel/Application/Admins"
)
```

---

Ghi chú:
- Các đoạn mã đã được trích gọn để minh hoạ trực tiếp; vui lòng xem file nguồn để biết ngữ cảnh đầy đủ.


