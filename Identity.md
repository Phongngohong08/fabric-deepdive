## Identity

Audience: Architects, Administrators, Developers

Tài liệu này tóm tắt khái niệm danh tính (Identity) trong Fabric và đính kèm các trích đoạn mã ngắn liên quan tới MSP, Principal, và chữ ký số.

### What is an Identity?

- Mọi tác nhân (peer, orderer, ứng dụng, admin, …) đều có danh tính số dưới dạng chứng chỉ X.509.
- Quyền truy cập được quyết định bởi danh tính và các thuộc tính kèm theo (principal: role, OU, org, …).
- Danh tính phải có thể kiểm chứng được bởi một nhà phát hành tin cậy. Trong Fabric, thành phần đó là MSP.

### PKI và vai trò của MSP

- PKI cung cấp hạ tầng chứng chỉ và khoá công khai/bí mật để xác thực và toàn vẹn thông điệp.
- MSP định nghĩa quy tắc xác thực danh tính hợp lệ cho một tổ chức (root/intermediate CAs, admins, OU, NodeOUs, TLS roots…).

Code minh hoạ (cấu hình MSP biên dịch từ `configtx.yaml`):

```96:144:vendor/github.com/hyperledger/fabric-protos-go-apiv2/msp/msp_config.pb.go
type FabricMSPConfig struct {
    Name string                     // MSP ID
    RootCerts [][]byte              // Danh sách CA gốc tin cậy
    IntermediateCerts [][]byte      // CA trung gian
    Admins [][]byte                 // Chứng chỉ admin của MSP
    RevocationList [][]byte         // CRLs (danh sách thu hồi)
    // SigningIdentity ... (thông tin danh tính ký mặc định)
    OrganizationalUnitIdentifiers []*FabricOUIdentifier // OU của MSP
    CryptoConfig *FabricCryptoConfig                    // Thuật toán băm/ký
    TlsRootCerts [][]byte                               // TLS root CAs
    TlsIntermediateCerts [][]byte                       // TLS intermediate CAs
    FabricNodeOus *FabricNodeOUs                        // Phân biệt Client/Peer/Orderer theo OU
}
```

Code minh hoạ (xây MSPConfig từ thư mục MSP):

```342:356:msp/configbuilder.go
fmspconf := &msp.FabricMSPConfig{
    Admins: admincert,                 // Danh tính admin
    RootCerts: cacerts,                // Root CA
    IntermediateCerts: intermediatecerts, // Intermediate CAs
    SigningIdentity: sigid,            // Khoá/ký mặc định
    Name: ID,                          // MSP ID
    OrganizationalUnitIdentifiers: ouis,
    RevocationList: crls,
    CryptoConfig: cryptoConfig,        // Thuật toán băm
    TlsRootCerts: tlsCACerts,
    TlsIntermediateCerts: tlsIntermediateCerts,
    FabricNodeOus: nodeOUs,            // Bật NodeOUs để phân vai trò theo OU
}
```

### Serialized Identity

- Khi trao đổi trên mạng, danh tính thường được tuần tự hoá kèm MSP ID.

Code minh hoạ (protobuf `SerializedIdentity`):

```27:39:vendor/github.com/hyperledger/fabric-protos-go-apiv2/msp/identities.pb.go
type SerializedIdentity struct {
    Mspid string  // MSP ID của danh tính
    IdBytes []byte // Bytes danh tính (X.509 PEM hoặc Idemix)
}
```

Code minh hoạ (serialize danh tính X.509 trong MSP):

```152:165:msp/identities.go
func NewSerializedIdentity(mspID string, certPEM []byte) ([]byte, error) {
    sId := &msp.SerializedIdentity{Mspid: mspID, IdBytes: certPEM}
    return proto.Marshal(sId)
}
```

### Principals và Signature Policies

- Principal là cách khớp danh tính với vai trò/quy tắc. Thường dùng trong chính sách chữ ký (endorsement, readers/writers…).

Code minh hoạ (cấu trúc SignaturePolicyEnvelope và MSPPrincipal):

```138:158:docs/source/policies.rst
message SignaturePolicyEnvelope { int32 version = 1; SignaturePolicy policy = 2; repeated MSPPrincipal identities = 3; }
message MSPPrincipal { enum Classification { ROLE = 0; ORGANIZATION_UNIT = 1; IDENTITY  = 2; } Classification principal_classification = 1; bytes principal = 2; }
message MSPRole { string msp_identifier = 1; enum MSPRoleType { MEMBER = 0; ADMIN = 1; CLIENT = 2; PEER = 3; } MSPRoleType role = 2; }
```

Code minh hoạ (tạo policy theo vai trò bằng DSL):

```61:76:common/policydsl/policydsl_builder.go
func SignedByMspPeer(mspId string) *cb.SignaturePolicyEnvelope { // Yêu cầu 1 chữ ký từ bất kỳ peer của MSP
    return signedByFabricEntity(mspId, mb.MSPRole_PEER)
}
```

### Xác minh chữ ký và vai trò SigningIdentity

- `SigningIdentity` có khả năng ký và xác minh chữ ký. Việc xác minh sử dụng khoá công khai trong chứng chỉ và tuân theo thuật toán trong `CryptoConfig`.

Code minh hoạ (serialize và sign/verify):

```208:224:msp/identities.go
func (id *identity) Serialize() ([]byte, error) { // tạo SerializedIdentity từ cert
    pb := &pem.Block{Bytes: id.cert.Raw, Type: "CERTIFICATE"}
    pemBytes := pem.EncodeToMemory(pb)
    sId := &msp.SerializedIdentity{Mspid: id.id.Mspid, IdBytes: pemBytes}
    return proto.Marshal(sId)
}

type signingidentity struct { identity; signer crypto.Signer }
// Sign(msg []byte) ([]byte, error)
// Verify(msg []byte, sig []byte) error
```

### Idemix (tùy chọn ẩn danh)

- Ngoài X.509, Fabric hỗ trợ Idemix để ẩn danh người ký. Danh tính Idemix cũng được nhúng vào `SerializedIdentity` qua `id_bytes`.

Code minh hoạ (protobuf danh tính Idemix):

```12:35:vendor/github.com/IBM/idemix/idemixmsp/identities.proto
message SerializedIdemixIdentity {
    bytes nym_x = 1; // public pseudonym X
    bytes nym_y = 2; // public pseudonym Y
    bytes ou   = 3; // OU
    bytes role = 4; // vai trò (ADMIN/MEMBER)
    bytes proof = 5; // bằng chứng mật mã
}
```

---

Ghi chú:
- Các trích dẫn đã lược bỏ import/chi tiết không cần thiết, tập trung vào ý chính.
- Tham khảo tệp nguồn đầy đủ trong repo để xem ngữ cảnh triển khai.


