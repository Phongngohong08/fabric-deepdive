## Membership Service Provider (MSP)

Audience: Architects, Administrators, Developers

Tài liệu này tóm tắt MSP theo phong cách kèm mã nguồn: vì sao cần MSP, MSP local vs channel, cấu trúc thư mục, cách load/parse và sử dụng trong runtime.

### Why do I need an MSP?

- Fabric là mạng permissioned: danh tính X.509 cần được “chấp nhận” bởi mạng để có vai trò và quyền. MSP là cơ chế biến danh tính thành vai trò hợp lệ (admin, peer, client, orderer) và được dùng để xác thực/ủy quyền khi ký, endorse, hoặc cấu hình.

### MSP domains (Local vs Channel)

- **Local MSP**: tồn tại trên filesystem của node hoặc client, xác định ai là admin/node user ở cấp cục bộ; dùng để ký và xác thực ngoài ngữ cảnh kênh.
- **Channel MSP**: được khai báo trong channel config; xác định thành viên và quyền ở cấp kênh; bản sao đồng bộ tới mọi node qua consensus.

Code minh hoạ (quản lý Local MSP và MSP manager theo kênh):

```78:123:msp/mgmt/mgmt.go
// GetLocalMSP returns the local msp (and creates it if it doesn't exist)
func GetLocalMSP(cryptoProvider bccsp.BCCSP) msp.MSP { /* loadLocalMSP(...) */ }
// GetManagerForChain returns the msp manager for the supplied chain (channel)
func GetManagerForChain(chainID string) msp.MSPManager { /* map[channel]MSPManager */ }
```

Code minh hoạ (khởi tạo Local MSP từ cấu hình thư mục MSP):

```92:122:msp/mgmt/mgmt.go
func loadLocalMSP(bccsp bccsp.BCCSP) msp.MSP {
    mspType := viper.GetString("peer.localMspType") // mặc định "fabric"
    mspInst, err := msp.New(newOpts, bccsp)          // tạo MSP instance
    mspInst, err = cache.New(mspInst)                // bọc cache cho Fabric MSP
    return mspInst
}
```

Code minh hoạ (thiết lập Local MSP bằng thư mục):

```130:142:msp/mgmt/mgmt_test.go
conf, _ := msp.GetLocalMspConfig(mspDir, nil, "SampleOrg")
_ = GetLocalMSP(cryptoProvider).Setup(conf)
_, _ = GetLocalMSP(cryptoProvider).GetDefaultSigningIdentity()
```

### MSP structure and configuration

- MSP dựa trên bộ thư mục: `cacerts`, `intermediatecerts`, `signcerts`, `keystore`, `tlscacerts`, `tlsintermediatecacerts`, `config.yaml` (NodeOUs), và CRL.
- Khi render vào channel config, MSP được đóng gói thành `FabricMSPConfig`.

Code minh hoạ (cấu trúc `FabricMSPConfig` – có chú thích một dòng):

```96:144:vendor/github.com/hyperledger/fabric-protos-go-apiv2/msp/msp_config.pb.go
type FabricMSPConfig struct {
    Name string                     // MSP ID
    RootCerts [][]byte              // CA gốc tin cậy
    IntermediateCerts [][]byte      // CA trung gian
    Admins [][]byte                 // Chứng chỉ admin
    RevocationList [][]byte         // CRLs (thu hồi)
    // SigningIdentity ...          // (tuỳ chọn) danh tính ký mặc định
    OrganizationalUnitIdentifiers []*FabricOUIdentifier // Danh sách OU của MSP
    CryptoConfig *FabricCryptoConfig                    // Thuật toán băm/ký
    TlsRootCerts [][]byte                               // TLS root CAs
    TlsIntermediateCerts [][]byte                       // TLS intermediate CAs
    FabricNodeOus *FabricNodeOUs                        // Bật phân vai trò theo OU (client/peer/admin/orderer)
}
```

Code minh hoạ (xây `MSPConfig` từ thư mục MSP):

```342:356:msp/configbuilder.go
fmspconf := &msp.FabricMSPConfig{ Admins: admincert, RootCerts: cacerts, IntermediateCerts: intermediatecerts, SigningIdentity: sigid, Name: ID, OrganizationalUnitIdentifiers: ouis, RevocationList: crls, CryptoConfig: cryptoConfig, TlsRootCerts: tlsCACerts, TlsIntermediateCerts: tlsIntermediateCerts, FabricNodeOus: nodeOUs }
return &msp.MSPConfig{Config: proto.Marshal(fmspconf), Type: int32(FABRIC)}
```

Code minh hoạ (parse MSP trong channel config và chuyển về struct runtime):

```544:604:vendor/github.com/hyperledger/fabric-config/configtx/msp.go
func getMSPConfig(configGroup *cb.ConfigGroup) (MSP, error) {
    // Unmarshal FabricMSPConfig, parse Root/Intermediates/Admins/CRLs/OU/TLS roots
    // Trả về đối tượng MSP (Name, RootCerts, Admins, NodeOUs, ...)
}
```

### Principals, NodeOUs và vai trò

- Vai trò (client/peer/admin/orderer) thường được gắn qua NodeOUs trong `config.yaml`, hoặc dùng trực tiếp trong `MSPPrincipal`/`MSPRole` của policies.

Ví dụ NodeOUs trong `config.yaml` (trích từ tài liệu):

```yaml
NodeOUs:
  Enable: true
  ClientOUIdentifier:
    Certificate: cacerts/ca.sampleorg-cert.pem
    OrganizationalUnitIdentifier: client
  PeerOUIdentifier:
    Certificate: cacerts/ca.sampleorg-cert.pem
    OrganizationalUnitIdentifier: peer
  AdminOUIdentifier:
    Certificate: cacerts/ca.sampleorg-cert.pem
    OrganizationalUnitIdentifier: admin
  OrdererOUIdentifier:
    Certificate: cacerts/ca.sampleorg-cert.pem
    OrganizationalUnitIdentifier: orderer
```

### MSP at runtime (deserialize/verify)

- MSPManager giữ nhiều MSP; cung cấp hàm `DeserializeIdentity` để từ `SerializedIdentity` khôi phục `Identity` và xác minh chữ ký.

Code minh hoạ (khởi tạo MSPManager với danh sách MSP):

```36:67:msp/mspmgrimpl.go
func (mgr *mspManagerImpl) Setup(msps []MSP) error { // Lưu map[MSPID]MSP, sắp xếp theo ProviderType, đánh dấu up=true }
```

---

Ghi chú:
- Các trích dẫn đã được gọn hoá để tập trung vào ý chính; hãy mở file nguồn để xem đầy đủ import và xử lý lỗi.


