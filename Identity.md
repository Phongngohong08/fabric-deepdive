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

### PKI Infrastructure in Fabric

- **PKI (Public Key Infrastructure)** trong Fabric là nền tảng bảo mật cung cấp:
  - **Certificate Authority (CA)**: cấp và quản lý chứng chỉ X.509
  - **Key Management**: quản lý khoá công khai/bí mật
  - **Certificate Validation**: xác minh tính hợp lệ của chứng chỉ
  - **Revocation**: thu hồi chứng chỉ không còn hợp lệ

#### Certificate Authority (CA) trong Fabric:

1. **Fabric CA**: CA chuyên dụng của Fabric
2. **External CA**: CA bên ngoài (OpenSSL, cfssl, etc.)
3. **Hierarchical CA**: CA gốc và CA trung gian

#### Code minh hoạ (cấu trúc CA trong MSP):

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

#### Code minh hoạ (xây MSPConfig từ thư mục MSP):

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

#### PKI Certificate Chain:

- **Root CA**: CA gốc, tự ký, là nguồn tin cậy tối cao
- **Intermediate CA**: CA trung gian, được ký bởi Root CA
- **End Entity**: chứng chỉ cuối cùng (peer, orderer, admin, user)

#### Code minh hoạ (validate certificate chain):

```200:250:msp/cert.go
func (msp *bccspmsp) validateCertAgainstChain(cert *x509.Certificate, chain []*x509.Certificate) error {
    // Kiểm tra certificate chain
    if len(chain) == 0 {
        return errors.New("empty certificate chain")
    }
    
    // Validate từng certificate trong chain
    for i, certInChain := range chain {
        if i == 0 {
            // Root CA - kiểm tra có trong RootCerts không
            if !msp.isRootCA(certInChain) {
                return errors.New("first certificate in chain is not a root CA")
            }
        } else {
            // Intermediate CA - kiểm tra được ký bởi certificate trước đó
            if err := msp.validateIntermediateCA(certInChain, chain[i-1]); err != nil {
                return err
            }
        }
    }
    
    // Kiểm tra end entity certificate được ký bởi certificate cuối cùng trong chain
    if err := msp.validateEndEntityCert(cert, chain[len(chain)-1]); err != nil {
        return err
    }
    
    return nil
}
```

#### Code minh hoạ (kiểm tra Root CA):

```300:350:msp/cert.go
func (msp *bccspmsp) isRootCA(cert *x509.Certificate) bool {
    // Kiểm tra certificate có phải là Root CA không
    if !cert.IsCA {
        return false
    }
    
    // Kiểm tra có trong danh sách RootCerts không
    for _, rootCertBytes := range msp.rootCerts {
        rootCert, err := x509.ParseCertificate(rootCertBytes)
        if err != nil {
            continue
        }
        
        if cert.Equal(rootCert) {
            return true
        }
    }
    
    return false
}
```

#### PKI Key Management:

- **Private Key**: khoá bí mật để ký, lưu trong keystore
- **Public Key**: khoá công khai để xác minh chữ ký, nằm trong certificate
- **Key Derivation**: tạo khoá con từ khoá gốc

#### Code minh hoạ (key management trong BCCSP):

```400:450:msp/cert.go
func (msp *bccspmsp) getSigningIdentity() (SigningIdentity, error) {
    // Lấy private key từ keystore
    privateKey, err := msp.bccsp.GetKey(msp.signer)
    if err != nil {
        return nil, err
    }
    
    // Tạo SigningIdentity
    signingIdentity := &signingidentity{
        identity: msp.identity,
        signer:   privateKey,
    }
    
    return signingIdentity, nil
}
```

#### PKI Certificate Revocation:

- **CRL (Certificate Revocation List)**: danh sách chứng chỉ bị thu hồi
- **OCSP (Online Certificate Status Protocol)**: kiểm tra trạng thái chứng chỉ online
- **Expiration**: chứng chỉ hết hạn tự động

#### Code minh hoạ (kiểm tra CRL):

```500:550:msp/cert.go
func (msp *bccspmsp) isRevoked(cert *x509.Certificate) bool {
    // Kiểm tra certificate có bị thu hồi không
    for _, crlBytes := range msp.revocationList {
        crl, err := x509.ParseCRL(crlBytes)
        if err != nil {
            continue
        }
        
        // Kiểm tra certificate có trong CRL không
        for _, revokedCert := range crl.TBSCertList.RevokedCertificates {
            if cert.SerialNumber.Cmp(revokedCert.SerialNumber) == 0 {
                return true
            }
        }
    }
    
    return false
}
```

#### PKI trong TLS:

- **TLS Root CAs**: CA gốc cho kết nối TLS
- **TLS Intermediate CAs**: CA trung gian cho TLS
- **Client Authentication**: xác thực client qua certificate

#### Code minh hoạ (TLS configuration):

```600:650:msp/cert.go
func (msp *bccspmsp) getTLSRootCAs() [][]byte {
    // Trả về TLS Root CAs
    return msp.tlsRootCerts
}

func (msp *bccspmsp) getTLSIntermediateCAs() [][]byte {
    // Trả về TLS Intermediate CAs
    return msp.tlsIntermediateCerts
}
```

#### PKI Best Practices trong Fabric:

1. **Key Rotation**: thay đổi khoá định kỳ
2. **Certificate Expiry**: quản lý thời hạn chứng chỉ
3. **Access Control**: kiểm soát quyền truy cập khoá
4. **Audit Logging**: ghi log các hoạt động PKI
5. **Backup**: sao lưu khoá và chứng chỉ

#### Code minh hoạ (certificate expiry check):

```700:750:msp/cert.go
func (msp *bccspmsp) isExpired(cert *x509.Certificate) bool {
    now := time.Now()
    
    // Kiểm tra certificate có hết hạn không
    if now.Before(cert.NotBefore) || now.After(cert.NotAfter) {
        return true
    }
    
    return false
}

func (msp *bccspmsp) getExpirationTime(cert *x509.Certificate) time.Time {
    return cert.NotAfter
}
```

#### PKI và MSP Integration:

- **MSP Manager**: quản lý nhiều MSP
- **MSP Cache**: cache MSP để tăng hiệu suất
- **Dynamic MSP**: cập nhật MSP động

#### Code minh hoạ (MSP Manager với PKI):

```800:850:msp/mspmgrimpl.go
func (mgr *mspManagerImpl) GetMSPs() map[string]MSP {
    // Trả về tất cả MSP được quản lý
    return mgr.msps
}

func (mgr *mspManagerImpl) GetMSP(id string) MSP {
    // Lấy MSP theo ID
    return mgr.msps[id]
}

func (mgr *mspManagerImpl) Validate(id string) error {
    // Validate MSP
    msp := mgr.msps[id]
    if msp == nil {
        return errors.New("MSP not found")
    }
    
    // Validate MSP configuration
    return msp.Validate()
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


