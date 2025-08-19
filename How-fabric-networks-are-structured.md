## How Fabric networks are structured

Audience: Architects, Administrators, Developers

Tài liệu này tóm tắt cấu trúc mạng Hyperledger Fabric ở mức khái niệm, đồng thời đính kèm các đoạn mã minh hoạ trực tiếp từ repo để bạn liên hệ giữa lý thuyết và triển khai.

### What is a blockchain network?

- **Network/Channel**: trong Fabric, khi nói tới “network” chúng ta thường ám chỉ tập hợp tổ chức, node, chính sách và quy trình xoay quanh một kênh (channel).
- **Ledger & Smart contract**: ứng dụng gửi proposal tới smart contract (chaincode) để tạo giao dịch; các giao dịch được order thành block và phân phối tới peer để ghi vào ledger.

### The sample network

- Mô tả :
  - **Tổ chức**: R1 và R2 là tổ chức peer; R0 là tổ chức orderer. Mỗi tổ chức có CA riêng (CA1, CA2, CA0) cấp danh tính cho admin, node, ứng dụng.
  - **Kênh**: `C1` có cấu hình `CC1` liệt kê ba tổ chức và các chính sách (Readers/Writers/Admins; mod_policy; capabilities...).
  - **Node**:
    - Peer `P1` của R1 và peer `P2` của R2 tham gia `C1` và trở thành anchor peers của tổ chức mình để bootstrap gossip.
    - Orderer `O` của R0 tham gia `C1` (ví dụ consenter id `1`).
  - **Ledger**: mỗi peer giữ bản sao ledger `L1` của `C1` gồm blockchain + world state; orderer chỉ giữ blockchain (không có state DB).
  - **Chaincode**: smart contract `S5` được cài trên `P1`, `P2` và được commit vào `C1` với một endorsement policy (ví dụ yêu cầu chữ ký của cả R1 và R2, hoặc số đông các org ứng dụng trên kênh).
  - **Ứng dụng**: `A1` (thuộc R1) và `A2` (thuộc R2) kết nối tới Gateway của peer trong tổ chức mình để thực hiện giao dịch trên `S5`.

- Luồng giao dịch end‑to‑end :
  1. Ứng dụng (ví dụ `A1`) kết nối gRPC tới Gateway trên một peer của R1. Gateway tự nội suy kế hoạch endorsement phù hợp dựa trên discovery và chính sách.
  2. Gateway gửi proposal tới các peer cần thiết (thường `P1` và `P2`). Mỗi peer chạy hàm chaincode `S5`, tạo `RW-set` và trả về phản hồi kèm chữ ký (endorsement).
  3. Gateway ghép các phản hồi hợp lệ thành một transaction envelope và trả cho ứng dụng ký (client signature).
  4. Ứng dụng gửi transaction đã ký tới orderer `O`. Orderer sắp xếp giao dịch, cắt block mới và phát block đó cho tất cả peer trong `C1`.
  5. Mỗi peer validate từng giao dịch trong block (kiểm chính sách endorsement, kiểm MVCC/phantom), đánh dấu hợp lệ/không hợp lệ, cập nhật world state và ghi block vào blockchain cục bộ. Event cam kết được phát về cho client.
  6. Từ thời điểm này, truy vấn tới `P1`/`P2` phản ánh trạng thái mới; `O` chỉ lưu chuỗi block để bảo toàn thứ tự giao dịch.

- Giải thích nhanh về **anchor peer**:
  - **Mục đích**: là điểm bootstrap để các peer của tổ chức khác biết cách kết nối vào gossip của tổ chức bạn. Mỗi org nên khai báo tối thiểu 1 anchor peer (có thể nhiều để HA).
  - **Đặc tính**: không có quyền đặc biệt về endorsement/commit; chỉ là địa chỉ được công bố trong channel config (host, port). Có thể trùng hoặc khác với leading peer.
  - **Tác động thực tế**: thiếu anchor peer vẫn có thể vận hành nội bộ từng org, nhưng peer giữa các org khó/không phát hiện nhau dẫn đến việc lan truyền block/metadata qua gossip bị chậm hoặc thất bại.
  - **Quản trị**: được lưu trong cấu hình kênh và có thể cập nhật bằng một channel config update; không cần cài đặt lại chaincode.

Code minh hoạ (kiểu dữ liệu kênh, tổ chức, policy, anchor peer khi tạo cấu hình):

```1:43:vendor/github.com/hyperledger/fabric-config/configtx/config.go
// Channel is a channel configuration.
type Channel struct {
    Consortium   string              // Tên consortium dùng để tạo kênh (tập các org được phép tham gia)
    Application  Application         // Cấu hình phần Application (org peer, policies, ACLs, capabilities)
    Orderer      Orderer             // Cấu hình ordering service cho kênh (loại, consenter set, policies)
    Consortiums  []Consortium        // Danh sách consortiums (dùng khi tạo system profile hoặc nhiều consortium)
    Capabilities []string            // Tập tính năng bật cho kênh (ví dụ V2_0)
    Policies     map[string]Policy   // Các policy cấp kênh (Readers/Writers/Admins/BlockValidation...)
    ModPolicy    string              // Chính sách kiểm soát việc sửa đổi mục này (thường là Admins)
}

// Policy is an expression used to define rules for access...
type Policy struct {
    Type      string // Kiểu policy: SIGNATURE hoặc IMPLICIT_META
    Rule      string // Biểu thức policy, ví dụ "OutOf(1, 'Org1.admin')" hoặc "MAJORITY Admins"
    ModPolicy string // Policy kiểm soát quyền sửa đổi policy này
}

// Organization ... includes AnchorPeers and OrdererEndpoints
type Organization struct {
    Name     string            // Tên hiển thị của tổ chức trong cấu hình kênh
    Policies map[string]Policy // Bộ policy cấp tổ chức (Admins/Readers/Writers/Endorsement...)
    MSP      MSP               // Cấu hình MSP (root certs, admins, OU, tlscacerts...)
    AnchorPeers      []Address // Danh sách anchor peers của org (host/port) để bootstrap gossip
    OrdererEndpoints []string  // (Cho org orderer) danh sách endpoint orderer mà org công bố
    ModPolicy        string    // Policy kiểm soát sửa đổi mục tổ chức này
}
```

### Creating the network (channel configuration)

- Kênh tồn tại **logic** sau khi có block cấu hình: danh sách tổ chức, MSP, policies, capabilities…
- Fabric parse cấu hình kênh thành các nhóm (`Application`, `Orderer`, `Consortiums`), khởi tạo MSP manager.

Code minh hoạ (parse cấu hình kênh và tạo MSP manager):

```83:128:common/channelconfig/channel.go
// NewChannelConfig creates a new ChannelConfig
func NewChannelConfig(channelGroup *cb.ConfigGroup, bccsp bccsp.BCCSP) (*ChannelConfig, error) {
    cc := &ChannelConfig{ protos: &ChannelProtos{} }
    if err := DeserializeProtoValuesFromGroup(channelGroup, cc.protos); err != nil {
        return nil, errors.Wrap(err, "failed to deserialize values")
    }
    channelCapabilities := cc.Capabilities()
    if err := cc.Validate(channelCapabilities); err != nil { return nil, err }
    mspConfigHandler := NewMSPConfigHandler(channelCapabilities.MSPVersion(), bccsp)
    var err error
    for groupName, group := range channelGroup.Groups {
        switch groupName {
        case ApplicationGroupKey:
            cc.appConfig, err = NewApplicationConfig(group, mspConfigHandler)
        case OrdererGroupKey:
            cc.ordererConfig, err = NewOrdererConfig(group, mspConfigHandler, channelCapabilities)
        case ConsortiumsGroupKey:
            cc.consortiumsConfig, err = NewConsortiumsConfig(group, mspConfigHandler)
        default:
            return nil, fmt.Errorf("Disallowed channel group: %s", group)
        }
        if err != nil { return nil, errors.Wrapf(err, "could not create channel %s sub-group config", groupName) }
    }
    if cc.mspManager, err = mspConfigHandler.CreateMSPManager(); err != nil { return nil, err }
    return cc, nil
}
```

### Certificate Authorities (CA) và MSP

- CA cấp chứng chỉ X.509 cho node, admin, ứng dụng; mapping sang tổ chức thông qua MSP.
- MSP định nghĩa tổ chức và xác thực chữ ký trong giao dịch/cấu hình.

Code minh hoạ (xử lý chứng chỉ trong `msp`):

```65:92:msp/cert.go
func isECDSASignedCert(cert *x509.Certificate) bool {
    return cert.SignatureAlgorithm == x509.ECDSAWithSHA1 ||
        cert.SignatureAlgorithm == x509.ECDSAWithSHA256 ||
        cert.SignatureAlgorithm == x509.ECDSAWithSHA384 ||
        cert.SignatureAlgorithm == x509.ECDSAWithSHA512
}
```

### Join nodes to the channel

- Peer lưu ledger (blockchain + state DB) cho mỗi channel đã join; orderer chỉ lưu blockchain.
- Khái niệm **anchor peers** giúp bootstrap gossip giữa các tổ chức.

Code minh hoạ (kiểu YAML anchor peers, dùng khi build profile với `configtxgen`):

```119:148:internal/configtxgen/genesisconfig/config.go
type Organization struct {
    Name string `yaml:"Name"`                 // Tên org trong profile configtx.yaml
    ID   string `yaml:"ID"`                   // MSP ID (ví dụ Org1MSP)
    MSPDir string `yaml:"MSPDir"`             // Đường dẫn thư mục MSP (certs, keys) của org
    Policies map[string]*Policy `yaml:"Policies"` // Tập policy của org dùng khi render profile
    AnchorPeers      []*AnchorPeer `yaml:"AnchorPeers"`      // Danh sách anchor peers cho Application org
    OrdererEndpoints []string      `yaml:"OrdererEndpoints"` // Danh sách endpoint cho Orderer org
}
type AnchorPeer struct { 
    Host string `yaml:"Host"` // Hostname/FQDN của anchor peer
    Port int    `yaml:"Port"` // Cổng TCP gRPC của anchor peer
}
```

Code minh hoạ (Application org parse `AnchorPeers`):

```34:67:common/channelconfig/applicationorg.go
// NewApplicationOrgConfig ...
func NewApplicationOrgConfig(id string, orgGroup *cb.ConfigGroup, mspConfig *MSPConfigHandler) (*ApplicationOrgConfig, error) {
    // ... deserialize values ...
}
// AnchorPeers returns the list of anchor peers of this Organization
func (aog *ApplicationOrgConfig) AnchorPeers() []*pb.AnchorPeer { // Trả về danh sách anchor peer của org trong cấu hình
    return aog.protos.AnchorPeers.AnchorPeers
}
```

Code minh hoạ (cập nhật anchor peer trong test-network):

```38:49:fabric-samples/test-network/scripts/setAnchorPeer.sh
# Modify the configuration to append the anchor peer 
jq '.channel_group.groups.Application.groups.'${CORE_PEER_LOCALMSPID}'.values += {"AnchorPeers":{...}}' \
  ${TEST_NETWORK_HOME}/channel-artifacts/${CORE_PEER_LOCALMSPID}config.json > \
  ${TEST_NETWORK_HOME}/channel-artifacts/${CORE_PEER_LOCALMSPID}modified_config.json
createConfigUpdate ${CHANNEL_NAME} ...anchors.tx
```

Code minh hoạ (gossip học anchor peers từ cấu hình):

```449:476:gossip/service/gossip_service.go
func (g *GossipService) updateAnchors(configUpdate ConfigUpdate) {
    // ... build joinChannelMessage from orgs' AnchorPeers and JoinChan(...)
}
```

### Install, approve, and commit a chaincode (lifecycle)

- Chu trình mới (v2.x): package → install → approve (mỗi org) → commit (trên kênh) → invoke.
- Chính sách endorsement được đặt trong định nghĩa chaincode và có thể cập nhật khi thêm tổ chức.

Code minh hoạ (CLI lifecycle các lệnh chính):

```19:46:internal/peer/lifecycle/chaincode/chaincode.go
const (
    lifecycleName = "_lifecycle"
    approveFuncName = "ApproveChaincodeDefinitionForMyOrg"
    commitFuncName  = "CommitChaincodeDefinition"
    checkCommitReadinessFuncName = "CheckCommitReadiness"
)
// Cmd() ... thêm các subcommand: Package, Install, ApproveForMyOrg, Commit, QueryCommitted...
```

Code minh hoạ (SCC xử lý Commit định nghĩa chaincode và kiểm tra approvals):

```585:616:core/chaincode/lifecycle/scc.go
cd := &ChaincodeDefinition{ Sequence: input.Sequence, ... }
approvals, err := i.SCC.Functions.CommitChaincodeDefinition(
    i.Stub.GetChannelID(), input.Name, cd, i.Stub, opaqueStates,
)
if !approvals[myOrg] { return nil, errors.Errorf("chaincode definition not agreed to by this org (%s)", i.SCC.OrgMSPID) }
```

### Using an application on the channel (Gateway)

- Từ Fabric v2.4, ứng dụng nên dùng Fabric Gateway SDK; peer có thể bật dịch vụ Gateway để thay ứng dụng thực hiện evaluate/endorse/submit/wait.

Code minh hoạ (peer bật Gateway và đăng ký server gRPC):

```900:918:internal/peer/node/start.go
if coreConfig.GatewayOptions.Enabled {
    gatewayServer := gateway.CreateServer(
        auth, discoveryService, peerInstance, &serverConfig.SecOpts, aclProvider,
        coreConfig.LocalMSPID, coreConfig.GatewayOptions, builtinSCCs,
    )
    gatewayprotos.RegisterGatewayServer(peerServer.Server(), gatewayServer)
}
```

Code minh hoạ (khởi tạo server Gateway):

```27:36:internal/pkg/gateway/gateway.go
// Server represents the GRPC server for the Gateway.
type Server struct {
    registry *registry
    commitFinder CommitFinder
    policy ACLChecker
    options config.Options
    ledgerProvider ledger.Provider
    getChannelConfig channelConfigGetter
}
```

### Storage/ledger trên peer

- Mỗi peer lưu trữ chaincodes và ledger (một ledger cho mỗi channel). StateDB là LevelDB hoặc CouchDB, cấu hình trong `core.yaml`.

Code minh hoạ (đọc cấu hình state DB trong peer):

```55:91:internal/peer/node/config.go
conf := &ledger.Config{ RootFSPath: ledgersDataRootDir, StateDBConfig: &ledger.StateDBConfig{ StateDatabase: viper.GetString("ledger.state.stateDatabase"), CouchDB: &ledger.CouchDBConfig{}, }, ... }
if conf.StateDBConfig.StateDatabase == ledger.CouchDB {
    conf.StateDBConfig.CouchDB = &ledger.CouchDBConfig{ Address: viper.GetString("ledger.state.couchDBConfig.couchDBAddress"), /* ... */ }
}
```

### Joining components to multiple channels

- Một peer có thể tham gia nhiều channel; mỗi channel có ledger độc lập (blockchain + world state + namespaces). Orderer nodes cũng có thể phục vụ nhiều kênh (mỗi kênh là một consenter set logic).

Code minh hoạ (mô tả cấu hình peer/orderer/channel trong kịch bản NWO):

```91:117:integration/nwo/network.go
// Peer defines a peer instance ... and the list of channels that the peer should be joined to.
type Peer struct { Name string; Organization string; Channels []*PeerChannel }
// PeerChannel ... whether the peer should be an anchor for the channel.
type PeerChannel struct { Name string; Anchor bool }
```

### Adding an organization to an existing channel

- Cập nhật channel config để thêm R3: quyết định quyền/policies; cập nhật định nghĩa chaincode (endorsement policy) và re-approve; sau đó org mới join peer vào kênh.
- Sau khi cập nhật, một block cấu hình mới (ví dụ `CC1.1`) được tạo ra.

Code minh hoạ (Application config chứa `AnchorPeers` trong cây cấu hình kênh):

```350:369:docs/source/configtx.rst
"Application":&ConfigGroup{ Groups:{ {{org_name}}:&ConfigGroup{ Values:{ "MSP":msp.MSPConfig, "AnchorPeers":peer.AnchorPeers }}}}
// ... Mỗi org mã hoá thêm AnchorPeers để các peer khác tổ chức có thể liên lạc qua gossip.
```

### Adding existing components to the newly joined channel

- Sau khi R3 được thêm và chaincode đã redefine/approve để bao gồm R3, có thể cài đặt S5 trên peer P3 và bắt đầu giao dịch; ứng dụng A3 cũng có thể sử dụng kênh.

---

Ghi chú:
- Các đoạn mã đã được trích gọn để tập trung vào ý chính và có thể bỏ qua một số dòng import/khởi tạo.
- Tham khảo các tệp nguồn đầy đủ trong repo để xem bối cảnh và toàn bộ triển khai.


