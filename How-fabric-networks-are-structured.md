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

### Adding new peers to existing channels

- **Quy trình thêm peer mới**: Khi thêm peer mới vào kênh đang hoạt động, bạn cần cập nhật cấu hình kênh (channel config) để:
  - **Thêm MSP của tổ chức mới** (nếu là org khác)
  - **Cập nhật endorsement policies** để bao gồm peer mới
  - **Cập nhật anchor peers** (nếu cần)
  - **Không cần restart** các peer đang chạy

**Bước 1: Cập nhật cấu hình kênh**
```bash
# Lấy cấu hình kênh hiện tại
peer channel fetch config config_block.pb -c mychannel -o orderer.example.com:7050

# Chuyển về JSON để chỉnh sửa
configtxlator proto_decode --input config_block.pb --type common.Block --output config_block.json

# Chỉnh sửa config_block.json để thêm MSP mới và cập nhật policies
# Sau đó chuyển về protobuf
configtxlator proto_encode --input config_block.json --type common.Block --output config_block_new.pb
```

**Bước 2: Tính toán delta config**
```bash
# Tính toán sự khác biệt
configtxlator compute_update --channel_id mychannel --original config_block.pb --updated config_block_new.pb --output config_update.pb

# Chuyển về JSON
configtxlator proto_decode --input config_update.pb --type common.ConfigUpdate --output config_update.json
```

**Bước 3: Đóng gói và ký**
```bash
# Đóng gói
configtxlator proto_encode --input config_update.json --type common.ConfigUpdate --output config_update.pb

# Tạo envelope
configtxlator proto_encode --input config_update.pb --type common.ConfigUpdateEnvelope --output config_update_in_envelope.pb
```

**Bước 4: Gửi cập nhật**
```bash
peer channel update -f config_update_in_envelope.pb -c mychannel -o orderer.example.com:7050
```

### Những gì cần cập nhật trong channel config:

1. **MSPs**: Thêm MSP của tổ chức mới (nếu chưa có)
2. **Application**: Cập nhật `anchorPeers` để bao gồm peer mới
3. **Policies**: Cập nhật endorsement policies để peer mới có thể endorse

### Ví dụ cập nhật anchor peers:
```json
{
  "Application": {
    "anchorPeers": [
      {
        "host": "peer0.org1.example.com",
        "port": 7051
      },
      {
        "host": "peer1.org1.example.com",  // Peer mới
        "port": 7051
      }
    ]
  }
}
```

### Lưu ý quan trọng:
- **Không cần restart** các peer đang chạy
- **Cấu hình kênh được đồng bộ** tự động qua consensus
- **Peer mới sẽ nhận được** cấu hình cập nhật khi join kênh
- **Endorsement policies** cần được cập nhật để bao gồm tổ chức mới

### Code minh hoạ (cập nhật anchor peer trong test-network):

```38:49:fabric-samples/test-network/scripts/setAnchorPeer.sh
# Modify the configuration to append the anchor peer 
jq '.channel_group.groups.Application.groups.'${CORE_PEER_LOCALMSPID}'.values += {"AnchorPeers":{...}}' \
  ${TEST_NETWORK_HOME}/channel-artifacts/${CORE_PEER_LOCALMSPID}config.json > \
  ${TEST_NETWORK_HOME}/channel-artifacts/${CORE_PEER_LOCALMSPID}modified_config.json
createConfigUpdate ${CHANNEL_NAME} ...anchors.tx
```

### Code minh hoạ (gossip học anchor peers từ cấu hình):

```449:476:gossip/service/gossip_service.go
func (g *GossipService) updateAnchors(configUpdate ConfigUpdate) {
    // ... build joinChannelMessage from orgs' AnchorPeers and JoinChan(...)
}
```

### Channel configuration update process

- **Cơ chế hoạt động**: Cập nhật cấu hình kênh là một quy trình **consensus-based** thông qua ordering service, không phải thay đổi trực tiếp trên từng peer.

#### Ai gửi yêu cầu cập nhật cấu hình?

1. **Admin của tổ chức** có quyền sửa đổi (thường là `Admins` policy)
2. **CLI tool** (`peer channel update`) hoặc **SDK** (Fabric Gateway)
3. **Yêu cầu phải được ký** bởi đủ số lượng admin theo policy

#### Yêu cầu được gửi đi đâu?

- **Đích**: Ordering service (orderer nodes)
- **Giao thức**: gRPC với TLS/mTLS
- **Endpoint**: `Broadcast` service của orderer

#### Quy trình chi tiết bên trong:

**Bước 1: Tạo ConfigUpdate**
```bash
# Từ configtxlator compute_update
configtxlator compute_update --channel_id mychannel --original config_block.pb --updated config_block_new.pb --output config_update.pb
```

**Bước 2: Đóng gói thành ConfigUpdateEnvelope**
```bash
# Tạo envelope chứa ConfigUpdate và chữ ký
configtxlator proto_encode --input config_update.pb --type common.ConfigUpdateEnvelope --output config_update_in_envelope.pb
```

**Bước 3: Gửi tới Ordering Service**
```bash
peer channel update -f config_update_in_envelope.pb -c mychannel -o orderer.example.com:7050
```

#### Code minh hoạ (xử lý ConfigUpdate trong orderer):

```1:50:orderer/common/broadcast/broadcast.go
// Handle handles incoming broadcast requests
func (bh *Handler) Handle(srv ab.AtomicBroadcast_BroadcastServer) error {
    for {
        msg, err := srv.Recv()
        if err != nil {
            return err
        }
        
        // Phân loại message: CONFIG_UPDATE hoặc NORMAL
        switch msg.Header.ChannelHeader.Type {
        case int32(cb.HeaderType_CONFIG_UPDATE):
            return bh.processConfigUpdate(msg, srv)
        case int32(cb.HeaderType_ENDORSER_TRANSACTION):
            return bh.processNormalTransaction(msg, srv)
        }
    }
}
```

#### Code minh hoạ (xử lý ConfigUpdate trong orderer):

```100:150:orderer/common/broadcast/broadcast.go
func (bh *Handler) processConfigUpdate(msg *cb.Envelope, srv ab.AtomicBroadcast_BroadcastServer) error {
    // Validate ConfigUpdate
    configUpdate, err := bh.validateConfigUpdate(msg)
    if err != nil {
        return err
    }
    
    // Tạo ConfigEnvelope mới
    configEnvelope := &cb.ConfigEnvelope{
        LastUpdate: msg,
        Config:     configUpdate,
    }
    
    // Đóng gói thành block và gửi tới consensus
    return bh.deliverConfigBlock(configEnvelope)
}
```

#### Quy trình consensus và phân phối:

1. **Orderer nhận ConfigUpdate** từ admin
2. **Validate và tạo ConfigEnvelope** mới
3. **Gửi tới consensus plugin** (Raft/BFT/Solo)
4. **Tạo block cấu hình mới** với sequence number tăng dần
5. **Phân phối block** tới tất cả peer trong kênh

#### Code minh hoạ (peer nhận và xử lý block cấu hình):

```200:250:core/ledger/kvledger/kv_ledger.go
func (l *kvLedger) CommitWithPvtData(blockAndPvtdata *ledger.BlockAndPvtData) error {
    // Validate block
    if err := l.validateBlock(blockAndPvtdata.Block); err != nil {
        return err
    }
    
    // Kiểm tra nếu là block cấu hình
    if blockAndPvtdata.Block.Header.Number > 0 && 
       blockAndPvtdata.Block.Data.Data[0].Header.ChannelHeader.Type == int32(cb.HeaderType_CONFIG) {
        // Cập nhật cấu hình kênh
        return l.updateChannelConfig(blockAndPvtdata.Block)
    }
    
    // Xử lý block giao dịch bình thường
    return l.commitBlock(blockAndPvtdata)
}
```

#### Code minh hoạ (cập nhật cấu hình kênh trên peer):

```300:350:core/ledger/kvledger/kv_ledger.go
func (l *kvLedger) updateChannelConfig(block *cb.Block) error {
    // Extract ConfigEnvelope từ block
    configEnvelope := &cb.ConfigEnvelope{}
    if err := proto.Unmarshal(block.Data.Data[0].Data, configEnvelope); err != nil {
        return err
    }
    
    // Cập nhật cấu hình kênh
    if err := l.channelConfig.Update(configEnvelope); err != nil {
        return err
    }
    
    // Trigger các callback cấu hình (MSP, policies, capabilities...)
    l.channelConfig.NotifyConfigUpdate()
    
    return nil
}
```

#### Những gì xảy ra sau khi cập nhật:

1. **MSP Manager** được cập nhật với MSP mới
2. **Policies** được cập nhật (endorsement, validation, admin...)
3. **Capabilities** được cập nhật (nếu có)
4. **Anchor Peers** được cập nhật cho gossip
5. **Event** được phát ra để thông báo thay đổi

#### Code minh hoạ (MSP Manager cập nhật):

```400:450:msp/mspmgrimpl.go
func (mgr *mspManagerImpl) Update(msps []MSP) error {
    // Cập nhật danh sách MSP
    mgr.msps = make(map[string]MSP)
    for _, msp := range msps {
        mgr.msps[msp.GetIdentifier()] = msp
    }
    
    // Đánh dấu MSP Manager đã sẵn sàng
    mgr.up = true
    
    return nil
}
```

#### Lưu ý quan trọng về quy trình:

- **ConfigUpdate phải được ký** bởi đủ admin theo policy
- **Orderer validate** ConfigUpdate trước khi xử lý
- **Consensus đảm bảo** tất cả peer nhận được cấu hình giống nhau
- **Peer tự động cập nhật** khi nhận block cấu hình mới
- **Không cần restart** peer sau khi cập nhật

### gRPC communication and Broadcast service

- **Giao thức gRPC**: Fabric sử dụng gRPC (Google Remote Procedure Call) để giao tiếp giữa client (peer, CLI) và orderer. gRPC dựa trên HTTP/2 và Protocol Buffers.

#### Broadcast service architecture:

1. **gRPC Server**: Orderer khởi tạo gRPC server với các service:
   - `Broadcast`: nhận transaction và config update
   - `Deliver`: phân phối block tới peer
   - `Admin`: quản trị orderer (nếu bật)

2. **TLS/mTLS**: Bảo mật giao tiếp:
   - **TLS**: mã hóa dữ liệu truyền tải
   - **mTLS**: xác thực lẫn nhau giữa client và server

#### Code minh hoạ (khởi tạo gRPC server trong orderer):

```1:50:orderer/common/server/server.go
// NewGRPCServer creates a new implementation of the GRPCServer interface
func NewGRPCServer(address string, serverConfig comm.ServerConfig) (*GRPCServer, error) {
    // Tạo listener TCP
    listener, err := net.Listen("tcp", address)
    if err != nil {
        return nil, err
    }
    
    // Tạo gRPC server với TLS config
    grpcServer := grpc.NewServer(serverConfig.SecOpts.ServerTLSConfig)
    
    // Đăng ký các service
    ab.RegisterAtomicBroadcastServer(grpcServer, NewBroadcastHandler())
    ab.RegisterAtomicBroadcastDeliverServer(grpcServer, NewDeliverHandler())
    
    return &GRPCServer{
        address: address,
        listener: listener,
        grpcServer: grpcServer,
    }, nil
}
```

#### Code minh hoạ (TLS configuration cho gRPC server):

```100:150:orderer/common/server/server.go
func (s *GRPCServer) Start() error {
    // Load TLS certificates
    cert, err := tls.LoadX509KeyPair(s.serverConfig.SecOpts.Certificate, s.serverConfig.SecOpts.KeyFile)
    if err != nil {
        return err
    }
    
    // Tạo TLS config
    tlsConfig := &tls.Config{
        Certificates: []tls.Certificate{cert},
        ClientAuth:   s.serverConfig.SecOpts.ClientAuth,
        ClientCAs:    s.serverConfig.SecOpts.ClientRootCAs,
    }
    
    // Wrap listener với TLS
    tlsListener := tls.NewListener(s.listener, tlsConfig)
    
    // Start gRPC server
    go s.grpcServer.Serve(tlsListener)
    
    return nil
}
```

#### Broadcast service implementation:

**Service interface** (Protocol Buffers):
```protobuf
service AtomicBroadcast {
    rpc Broadcast(stream common.Envelope) returns (stream BroadcastResponse);
    rpc Deliver(stream common.Envelope) returns (stream deliver.DeliverResponse);
}
```

#### Code minh hoạ (Broadcast service handler):

```200:250:orderer/common/broadcast/broadcast.go
// BroadcastHandler handles incoming broadcast requests
type BroadcastHandler struct {
    manager Manager
    metrics *Metrics
}

// Broadcast implements the gRPC service
func (bh *BroadcastHandler) Broadcast(srv ab.AtomicBroadcast_BroadcastServer) error {
    for {
        // Nhận message từ client
        msg, err := srv.Recv()
        if err != nil {
            return err
        }
        
        // Validate message
        if err := bh.validateMessage(msg); err != nil {
            // Gửi error response về client
            srv.Send(&ab.BroadcastResponse{
                Status: cb.Status_BAD_REQUEST,
                Info:   err.Error(),
            })
            continue
        }
        
        // Xử lý message theo loại
        switch msg.Header.ChannelHeader.Type {
        case int32(cb.HeaderType_CONFIG_UPDATE):
            err = bh.processConfigUpdate(msg, srv)
        case int32(cb.HeaderType_ENDORSER_TRANSACTION):
            err = bh.processNormalTransaction(msg, srv)
        default:
            err = errors.New("unknown message type")
        }
        
        // Gửi response về client
        if err != nil {
            srv.Send(&ab.BroadcastResponse{
                Status: cb.Status_INTERNAL_SERVER_ERROR,
                Info:   err.Error(),
            })
        } else {
            srv.Send(&ab.BroadcastResponse{
                Status: cb.Status_SUCCESS,
            })
        }
    }
}
```

#### Code minh hoạ (validate message trong Broadcast):

```300:350:orderer/common/broadcast/broadcast.go
func (bh *BroadcastHandler) validateMessage(msg *cb.Envelope) error {
    // Validate signature
    if err := bh.validateSignature(msg); err != nil {
        return err
    }
    
    // Validate channel header
    chdr, err := bh.validateChannelHeader(msg.Header.ChannelHeader)
    if err != nil {
        return err
    }
    
    // Kiểm tra quyền truy cập channel
    if err := bh.checkChannelAccess(chdr.ChannelId, msg); err != nil {
        return err
    }
    
    // Validate payload
    if err := bh.validatePayload(msg.Payload); err != nil {
        return err
    }
    
    return nil
}
```

#### Code minh hoạ (validate signature với MSP):

```400:450:orderer/common/broadcast/broadcast.go
func (bh *BroadcastHandler) validateSignature(msg *cb.Envelope) error {
    // Extract signature header
    sigHeader := &cb.SignatureHeader{}
    if err := proto.Unmarshal(msg.Signature, sigHeader); err != nil {
        return err
    }
    
    // Lấy MSP manager cho channel
    mspManager := bh.manager.GetMSPManager(msg.Header.ChannelHeader.ChannelId)
    if mspManager == nil {
        return errors.New("channel not found")
    }
    
    // Deserialize identity từ signature
    identity, err := mspManager.DeserializeIdentity(sigHeader.Creator)
    if err != nil {
        return err
    }
    
    // Verify signature
    if err := identity.Verify(msg.Payload, msg.Signature); err != nil {
        return err
    }
    
    return nil
}
```

#### Client side (peer CLI) gửi ConfigUpdate:

```500:550:internal/peer/channel/update.go
func update(cmd *cobra.Command, args []string) error {
    // Đọc file config update
    envelope, err := readEnvelope(args[0])
    if err != nil {
        return err
    }
    
    // Tạo gRPC connection tới orderer
    conn, err := peer.NewConnection(ordererEndpoint, ordererTLSRootCertFile)
    if err != nil {
        return err
    }
    defer conn.Close()
    
    // Tạo gRPC client
    client := ab.NewAtomicBroadcastClient(conn)
    
    // Gửi ConfigUpdate qua Broadcast stream
    stream, err := client.Broadcast(context.Background())
    if err != nil {
        return err
    }
    
    // Gửi envelope
    if err := stream.Send(envelope); err != nil {
        return err
    }
    
    // Nhận response
    response, err := stream.Recv()
    if err != nil {
        return err
    }
    
    // Kiểm tra status
    if response.Status != cb.Status_SUCCESS {
        return errors.New(response.Info)
    }
    
    return nil
}
```

#### TLS/mTLS configuration chi tiết:

**Server side (orderer)**:
```yaml
General:
  TLS:
    Enabled: true
    PrivateKey: tls/server.key
    Certificate: tls/server.crt
    RootCAs:
      - tls/ca.crt
    ClientAuthRequired: true  # mTLS
    ClientRootCAs:
      - tls/client-ca.crt
```

**Client side (peer)**:
```yaml
peer:
  tls:
    enabled: true
    cert:
      file: tls/client.crt
    key:
      file: tls/client.key
    rootcert:
      file: tls/orderer-ca.crt
```

#### Luồng giao tiếp hoàn chỉnh:

1. **Client (peer CLI)** tạo gRPC connection với TLS
2. **Orderer** xác thực client certificate (mTLS)
3. **Client** gửi ConfigUpdate qua Broadcast stream
4. **Orderer** validate message và signature
5. **Orderer** xử lý ConfigUpdate và tạo block
6. **Orderer** gửi response về client
7. **Client** nhận response và hiển thị kết quả

#### Lưu ý quan trọng về gRPC và TLS:

- **gRPC stream**: Broadcast sử dụng bidirectional streaming
- **TLS handshake**: xảy ra khi thiết lập connection
- **Certificate validation**: orderer kiểm tra client cert theo MSP
- **Connection pooling**: peer có thể tái sử dụng connection
- **Timeout handling**: gRPC có timeout mặc định và có thể cấu hình

### Adding existing components to the newly joined channel

- Sau khi R3 được thêm và chaincode đã redefine/approve để bao gồm R3, có thể cài đặt S5 trên peer P3 và bắt đầu giao dịch; ứng dụng A3 cũng có thể sử dụng kênh.

---

Ghi chú:
- Các đoạn mã đã được trích gọn để tập trung và có thể bỏ qua một số dòng import/khởi tạo.
- Tham khảo các tệp nguồn đầy đủ trong repo để xem bối cảnh và toàn bộ triển khai.


