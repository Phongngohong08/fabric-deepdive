## The Ordering Service

Audience: Architects, Ordering service admins, Channel creators

Tài liệu này tóm tắt vai trò orderer, cách tham gia vào luồng giao dịch, và các trích đoạn mã nguồn: cấu hình `BatchTimeout/BatchSize`, đăng ký dịch vụ Broadcast/Deliver, và ghi chú về Raft.

### What is ordering?

- Orderer sắp xếp (order) các giao dịch đã được endorse, cắt khối (block cutting), và phân phối block tới các peer trong kênh. Fabric dùng đồng thuận xác định → block là chung cuộc (final), không fork.

### Orderer và cấu hình kênh

- Orderer xử lý giao dịch cấu hình (config tx) để kiểm chính sách/ACL trước khi phát khối cấu hình tới peers.

### Cấu hình BatchTimeout/BatchSize

Code minh hoạ (trong `configtx.yaml`):

```178:199:fabric-samples/test-network/configtx/configtx.yaml
Orderer: &OrdererDefaults
  BatchTimeout: 2s                     # thời gian tối đa chờ cắt khối
  BatchSize:
    MaxMessageCount: 10                # số tx tối đa trong block
    AbsoluteMaxBytes: 99 MB            # kích thước tuyệt đối
    PreferredMaxBytes: 512 KB          # kích thước ưa thích
```

Code minh hoạ (đưa vào ConfigGroup qua protobuf):

```119:134:common/channelconfig/util_test.go
config.ChannelGroup.Groups[OrdererGroupKey].Values[BatchSizeKey] = &cb.ConfigValue{ Value: protoutil.MarshalOrPanic(&ab.BatchSize{ MaxMessageCount: 65535, AbsoluteMaxBytes: 1024000000, PreferredMaxBytes: 1024000000 }), ModPolicy: AdminsPolicyKey }
config.ChannelGroup.Groups[OrdererGroupKey].Values[BatchTimeoutKey] = &cb.ConfigValue{ Value: protoutil.MarshalOrPanic(&ab.BatchTimeout{ Timeout: "2s" }), ModPolicy: AdminsPolicyKey }
```

### Khởi động orderer và đăng ký dịch vụ Atomic Broadcast

- Orderer mở gRPC server và đăng ký `AtomicBroadcast` gồm hai RPC: `Broadcast` (ứng dụng gửi tx) và `Deliver` (client/peer nhận block).

Code minh hoạ (main khởi động server và đăng ký dịch vụ):

```197:235:orderer/common/server/main.go
server := NewServer(manager, metricsProvider, &conf.Debug, conf.General.Authentication.TimeWindow, mutualTLS, conf.General.Authentication.NoExpirationChecks)
throttlingWrapper := &ThrottlingAtomicBroadcast{ /* rate limit config */, AtomicBroadcastServer: server }
ab.RegisterAtomicBroadcastServer(grpcServer.Server(), throttlingWrapper)
_ = grpcServer.Start()
```

Code minh hoạ (tạo `server` gồm handler Broadcast/Deliver):

```84:103:orderer/common/server/server.go
func NewServer(r *multichannel.Registrar, metricsProvider metrics.Provider, debug *localconfig.Debug, timeWindow time.Duration, mutualTLS bool, expirationCheckDisabled bool) ab.AtomicBroadcastServer {
    s := &server{
        dh: deliver.NewHandler(deliverSupport{Registrar: r}, timeWindow, mutualTLS, deliver.NewMetrics(metricsProvider), expirationCheckDisabled),
        bh: &broadcast.Handler{ SupportRegistrar: broadcastSupport{Registrar: r}, Metrics: broadcast.NewMetrics(metricsProvider) },
        debug: debug, Registrar: r,
    }
    return s
}
```

Code minh hoạ (RPC triển khai):

```167:183:orderer/common/server/server.go
// Broadcast receives a stream of messages from a client for ordering
func (s *server) Broadcast(srv ab.AtomicBroadcast_BroadcastServer) error { return s.bh.Handle(&broadcastMsgTracer{ AtomicBroadcast_BroadcastServer: srv, msgTracer: msgTracer{ debug: s.debug, function: "Broadcast" } }) }
```

```185:218:orderer/common/server/server.go
// Deliver sends a stream of blocks to a client after ordering
func (s *server) Deliver(srv ab.AtomicBroadcast_DeliverServer) error { /* policy check và gọi s.dh.Handle(...) để stream block */ }
```

### Orderers trong luồng giao dịch

1) Sau khi Gateway thu thập đủ endorsements, ứng dụng gửi envelope tới `Broadcast` của orderer.
2) Orderer sắp xếp, cắt block theo `BatchTimeout/BatchSize`, ghi vào ledger của orderer, phát block tới peers (hoặc peers nhận qua gossip nếu offline).
3) Peers validate và commit (xem tài liệu "Peers").

### Consensus Mechanism and Transaction Flow

- **Cơ chế đồng thuận**: Orderer sử dụng consensus plugin để đảm bảo tất cả node đồng ý về thứ tự giao dịch và nội dung block. Fabric hỗ trợ nhiều loại consensus: Raft, BFT, Solo.

#### Luồng xử lý transaction trong orderer:

```
1. Gateway P1 thu thập đủ endorsements theo policy
2. Ứng dụng ký transaction và gửi về Gateway P1
3. Gateway P1 gửi transaction tới orderer (Broadcast service)
4. Orderer xử lý đồng thuận và tạo block
5. Orderer phân phối block tới tất cả peer
```

#### Consensus plugin interface:

```go
// Consensus plugin interface
type Consensus interface {
    Start() error
    Stop() error
    HandleMessage(sender uint64, msg *orderer.ConsensusMessage) error
}
```

#### Code minh hoạ (Broadcast service nhận transaction):

```200:250:orderer/common/broadcast/broadcast.go
func (bh *BroadcastHandler) Handle(srv ab.AtomicBroadcast_BroadcastServer) error {
    for {
        // Nhận message từ client
        msg, err := srv.Recv()
        if err != nil {
            return err
        }
        
        // Validate message
        if err := bh.validateMessage(msg); err != nil {
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

#### Code minh hoạ (xử lý transaction thường):

```300:350:orderer/common/broadcast/broadcast.go
func (bh *BroadcastHandler) processNormalTransaction(msg *cb.Envelope, srv ab.AtomicBroadcast_BroadcastServer) error {
    // Lấy channel từ message
    channelID := msg.Header.ChannelHeader.ChannelId
    
    // Lấy channel support
    support := bh.SupportRegistrar.GetChain(channelID)
    if support == nil {
        return errors.New("channel not found")
    }
    
    // Gửi transaction tới consensus plugin
    if err := support.Order(msg, 0); err != nil {
        return err
    }
    
    return nil
}
```

#### Block cutting logic:

```400:450:orderer/common/blockcutter/blockcutter.go
func (b *blockCutter) Ordered(msg *cb.Envelope) ([]*cb.Envelope, bool) {
    // Kiểm tra kích thước block hiện tại
    messageSizeBytes := messageSizeBytes(msg)
    
    // Nếu vượt quá PreferredMaxBytes, cắt block
    if b.messageCount > 0 && (b.messageBytes+messageSizeBytes) > b.preferredMaxBytes {
        return b.cut(), true
    }
    
    // Thêm message vào batch hiện tại
    b.messageCount++
    b.messageBytes += messageSizeBytes
    b.pending = append(b.pending, msg)
    
    // Kiểm tra MaxMessageCount
    if b.messageCount >= b.maxMessageCount {
        return b.cut(), true
    }
    
    return nil, false
}

func (b *blockCutter) cut() []*cb.Envelope {
    if len(b.pending) == 0 {
        return nil
    }
    
    batch := b.pending
    b.pending = nil
    b.messageCount = 0
    b.messageBytes = 0
    
    return batch
}
```

#### Raft consensus implementation:

```500:550:orderer/consensus/etcdraft/chain.go
func (c *Chain) Start() {
    // Khởi tạo Raft node
    c.Node = raft.StartNode(c.config, c.raftPeers)
    
    // Bắt đầu xử lý message
    go c.serveRequest()
    go c.serveRaft()
}

func (c *Chain) serveRequest() {
    for {
        select {
        case msg := <-c.submitC:
            // Xử lý transaction từ broadcast
            if err := c.Node.Propose(context.Background(), msg); err != nil {
                logger.Errorf("Failed to propose message: %s", err)
            }
        case <-c.doneC:
            return
        }
    }
}

func (c *Chain) serveRaft() {
    for {
        select {
        case <-c.Node.Ready():
            // Xử lý Raft events
            c.processRaftEvents()
        case <-c.doneC:
            return
        }
    }
}
```

#### Block creation và commit:

```600:650:orderer/consensus/etcdraft/chain.go
func (c *Chain) processRaftEvents() {
    // Xử lý committed entries
    for _, entry := range c.Node.CommittedEntries {
        if entry.Type == raft.EntryNormal {
            // Tạo block từ committed entries
            block := c.createBlock(entry.Data)
            
            // Gửi block tới deliver service
            c.deliverBlock(block)
        }
    }
}

func (c *Chain) createBlock(data []byte) *cb.Block {
    // Tạo block header
    header := &cb.BlockHeader{
        Number:       c.lastBlock.Header.Number + 1,
        PreviousHash: c.lastBlock.Header.Hash(),
        DataHash:     protoutil.BlockDataHash(c.lastBlock.Data),
    }
    
    // Tạo block
    block := &cb.Block{
        Header: header,
        Data: &cb.BlockData{
            Data: [][]byte{data},
        },
    }
    
    // Cập nhật last block
    c.lastBlock = block
    
    return block
}
```

#### Deliver service phân phối block:

```700:750:orderer/common/deliver/deliver.go
func (dh *DeliverHandler) Handle(srv ab.AtomicBroadcast_DeliverServer) error {
    for {
        // Nhận request từ client
        request, err := srv.Recv()
        if err != nil {
            return err
        }
        
        // Validate request
        if err := dh.validateRequest(request); err != nil {
            return err
        }
        
        // Lấy channel support
        support := dh.Registrar.GetChain(request.ChannelId)
        if support == nil {
            return errors.New("channel not found")
        }
        
        // Stream block tới client
        if err := dh.streamBlocks(srv, support, request); err != nil {
            return err
        }
    }
}

func (dh *DeliverHandler) streamBlocks(srv ab.AtomicBroadcast_DeliverServer, support ChannelSupport, request *ab.DeliverEnvelope) error {
    // Lấy block từ ledger
    block, err := support.GetBlock(request.SpecifiedNumber)
    if err != nil {
        return err
    }
    
    // Gửi block tới client
    if err := srv.Send(&ab.DeliverResponse{
        Type: &ab.DeliverResponse_Block{
            Block: block,
        },
    }); err != nil {
        return err
    }
    
    return nil
}
```

#### Consensus state management:

```800:850:orderer/consensus/etcdraft/chain.go
func (c *Chain) processLeaderChange(leader uint64) {
    if leader == c.Node.ID() {
        // Node này trở thành leader
        c.isLeader = true
        logger.Infof("Node %d became leader", c.Node.ID())
        
        // Bắt đầu xử lý transaction
        go c.processTransactions()
    } else {
        // Node này trở thành follower
        c.isLeader = false
        logger.Infof("Node %d became follower, leader is %d", c.Node.ID(), leader)
    }
}

func (c *Chain) processTransactions() {
    ticker := time.NewTicker(c.batchTimeout)
    defer ticker.Stop()
    
    for {
        select {
        case <-ticker.C:
            // Cắt block theo timeout
            if len(c.pendingTransactions) > 0 {
                c.cutBlock()
            }
        case <-c.doneC:
            return
        }
    }
}
```

#### Lưu ý quan trọng về consensus:

- **Leader election**: Raft tự động bầu leader, leader xử lý transaction
- **Log replication**: Leader replicate log entries tới followers
- **Quorum**: Cần đa số node đồng ý để commit
- **Fault tolerance**: Hệ thống vẫn hoạt động khi một số node lỗi
- **Block finality**: Block được commit là final, không thể thay đổi

### Cơ chế đồng thuận của Fabric (lý thuyết + ví dụ)

#### Lý thuyết

- **Hai tầng tách biệt**:
  - Tầng endorsement (ở peer): ứng dụng gọi chaincode, thu thập chữ ký theo chính sách. Tầng này không sắp xếp thứ tự giao dịch toàn cục.
  - Tầng ordering (ở orderer): sắp xếp giao dịch, cắt block, đồng thuận thứ tự. Orderer không thực thi chaincode cũng như không đánh giá chính sách endorsement khi order; việc validate chi tiết diễn ra ở peer khi commit (endorsement policy, MVCC read-write set, ACL…).
- **Plugin đồng thuận**: Fabric dùng kiến trúc plugin. Sản xuất hiện nay dùng `etcdraft` (CFT). Ở Fabric v3 có SmartBFT (BFT). Các cơ chế cũ như Solo/Kafka đã bị loại bỏ.
- **Finality và liveness**: Khi đạt quorum, block được commit mang tính chung cuộc (không fork). Khả dụng phụ thuộc còn đa số node; nếu mất quorum, kênh tạm thời không thể thêm block mới.
- **Điều chỉnh độ trễ/throughput**: `BatchTimeout/BatchSize` tác động kích thước/độ trễ block; tham số Raft như `TickInterval`, `ElectionTick`, `HeartbeatTick` tác động bầu leader và nhịp heartbeat.

#### Ví dụ cấu hình etcdraft (kênh mẫu)

- Chọn plugin đồng thuận cho orderer:

```255:260:fabric-samples/test-network/configtx/configtx.yaml
Orderer: &OrdererDefaults
  # Orderer Type: The orderer implementation to start.
  # Available types are "etcdraft" and "BFT".
  OrdererType: etcdraft
```

- Khai báo consenter set (3 node) và tuỳ chọn Raft mặc định cho kênh:

```352:365:fabric-samples/test-network/configtx/configtx.yaml
    Consenters:
      - Host: raft0.example.com
        Port: 7050
        ClientTLSCert: path/to/ClientTLSCert0
        ServerTLSCert: path/to/ServerTLSCert0
      - Host: raft1.example.com
        Port: 7050
        ClientTLSCert: path/to/ClientTLSCert1
        ServerTLSCert: path/to/ServerTLSCert1
      - Host: raft2.example.com
        Port: 7050
        ClientTLSCert: path/to/ClientTLSCert2
        ServerTLSCert: path/to/ServerTLSCert2
```

```366:391:fabric-samples/test-network/configtx/configtx.yaml
    Options:
      TickInterval: 500ms
      ElectionTick: 10
      HeartbeatTick: 1
      MaxInflightBlocks: 5
      SnapshotIntervalSize: 16 MB
```

- Diễn giải nhanh:
  - Nhịp heartbeat ≈ TickInterval × HeartbeatTick = 500ms.
  - Timeout bầu cử ≈ TickInterval × ElectionTick = 5s. Follower không nhận heartbeat trong ~5s sẽ mở bầu cử.
  - MaxInflightBlocks giới hạn số AppendEntries đang bay trong giai đoạn replicate lạc quan.
  - SnapshotIntervalSize kiểm soát kích thước dữ liệu giữa các lần snapshot log.

#### Ví dụ luồng giao dịch end-to-end (3 orderer, quorum=2)

1) Ứng dụng gửi envelope tới bất kỳ orderer trong consenter set qua RPC `Broadcast` (xem minh hoạ đoạn `broadcast.Handle` và `processNormalTransaction` ở trên).
2) Orderer nhận sẽ chuyển tiếp tới leader hiện tại của kênh (định tuyến nội bộ). Leader đề xuất (Propose) entry tới cụm Raft, các follower ghi nhận và trả ACK.
3) Khi đạt quorum ghi (ví dụ 2/3), entry được coi là committed. Logic tạo block (block cutter) gom các envelope theo `BatchSize/BatchTimeout` và tạo block.
4) Block được phát qua RPC `Deliver` tới client/peer. Peer chạy bước validate (endorsement policy, MVCC) rồi commit vào ledger và state DB. Ứng dụng nên lắng nghe commit event từ peer để xác nhận hoàn tất.
5) Sự cố leader: nếu leader sập, follower mất heartbeat ~5s sẽ bầu cử; leader mới tiếp tục replicate. Miễn còn ≥2 node sẵn sàng, kênh vẫn tiến triển.

### Raft (khuyến nghị)

- Mỗi kênh vận hành một cụm Raft riêng (consenter set). Một leader được bầu động, replicate log entries tới followers; khi đạt quorum → commit.
- Ưu điểm so với Kafka: cấu hình/triển khai đơn giản hơn, dễ phân tán nhiều tổ chức, được hỗ trợ trong cộng đồng Fabric.

### Phân tích chuyên sâu về Raft

- **Mô hình leader-follower và CFT**: Trong mỗi kênh, một leader được bầu từ tập consenter. Leader nhận log entries (giao dịch/config) và replicate tới followers. Hệ thống chịu lỗi dừng (Crash Fault Tolerant) miễn là còn đa số node (quorum). Ví dụ: 3 node chịu lỗi 1; 5 node chịu lỗi 2. Triển khai HA nên phân tán các orderer qua nhiều DC/vùng.
- **So với Kafka**: Cùng mô hình leader-follower và CFT ở mức chức năng, nhưng Raft dễ vận hành hơn (không cần Kafka/ZooKeeper, ít thành phần, không lệ thuộc phiên bản Kafka). Với Kafka, thường một tổ chức vận hành cụm Kafka → giảm mức phân quyền thực tế; Raft cho phép mỗi tổ chức có orderer riêng trong consenter set → phân quyền cao hơn. Fabric 2.x đã ngừng khuyến nghị Kafka và hướng tới BFT trong tương lai, Raft là bước đệm.
- **Cảnh báo về ack**: Giống Solo/Kafka, có thể mất giao dịch sau khi client nhận ack nếu leader crash đúng thời điểm. Ứng dụng phải lắng nghe commit event từ peer để xác nhận tính hợp lệ và xử lý timeout: có thể gửi lại hoặc thu thập lại endorsements tuỳ mô hình nghiệp vụ.

#### Khái niệm cốt lõi

- **Log entry/replicated log**: Đơn vị công việc là log entry. Log được coi là nhất quán khi đa số thành viên đồng ý thứ tự và nội dung.
- **Consenter set**: Tập orderer tham gia đồng thuận cho một kênh và nhận log replicate.
- **FSM (Finite-State Machine)**: Mỗi orderer áp dụng log theo cùng trình tự để trạng thái quyết định khối là tất định.
- **Quorum**: Đa số của consenter set. Thiếu quorum → cụm không thể đọc/ghi commit log mới trên kênh.
- **Leader/Follower**: Leader nhận và replicate log, quyết định commit khi đạt quorum; follower replicate quyết định và phát hiện mất heartbeat để kích hoạt bầu leader.

#### Raft trong luồng giao dịch của Fabric

- Mỗi kênh chạy một instance Raft riêng → mỗi kênh có leader riêng và có thể mở rộng/giảm node theo từng kênh (thêm/bớt từng node một lần). Orderer nhận tx sẽ tự định tuyến tới leader hiện tại của kênh nên ứng dụng/peer không cần biết leader là ai.
- Sau khi kiểm tra hợp lệ ở orderer, các tx được sắp xếp, đóng block, đạt đồng thuận và phân phối qua Deliver tới peer (như các đoạn code minh hoạ ở trên trong `broadcast`, `blockcutter`, `deliver`, và `etcdraft/chain`).

#### Bầu leader (leader election)

- Node có ba trạng thái: follower → candidate → leader. Ban đầu là follower; nếu không nhận log/heartbeat trong timeout, tự nâng cấp thành candidate và xin phiếu. Đạt quorum phiếu sẽ trở thành leader.
- Leader duy trì heartbeat; follower mất heartbeat sẽ khởi động bầu cử mới để đảm bảo sẵn sàng.

#### Snapshot và phục hồi

- Raft dùng snapshot để giới hạn dung lượng log trên đĩa. Khi một replica bị tụt hậu khởi động lại, nó sẽ nhận block snapshot gần nhất và đồng bộ phần thiếu qua Deliver và replication thường.
- Ví dụ: R1 ở block 100, leader ở block 196, snapshot ~20 block. R1 nhận block 180 từ leader làm mốc, sau đó yêu cầu Deliver 101→180, rồi nhận 181→196 qua replication.

#### Cấu hình thực tế (orderer.yaml)

- **Đường dẫn WAL/Snapshot cho etcdraft**:

```328:338:sampleconfig/orderer.yaml
Consensus:
  # The allowed key-value pairs here depend on consensus plugin. It is opaque
  # to orderer, and completely up to consensus implementation to make use of.

  # WALDir specifies the location at which Write Ahead Logs for etcd/raft are
  # stored. Each channel will have its own subdir named after channel ID.
  WALDir: /var/hyperledger/production/orderer/etcdraft/wal

  # SnapDir specifies the location at which snapshots for etcd/raft are
  # stored. Each channel will have its own subdir named after channel ID.
  SnapDir: /var/hyperledger/production/orderer/etcdraft/snapshot
```

- **Thiết lập kênh nội bộ `Cluster` giữa các orderer** (mTLS, cổng nghe riêng, chính sách replicate):

```86:117:sampleconfig/orderer.yaml
  # Cluster settings for ordering service nodes that communicate with other ordering service nodes
  # such as Raft based ordering service.
  Cluster:
    # SendBufferSize is the maximum number of messages in the egress buffer.
    SendBufferSize: 100
    # ClientCertificate / ClientPrivateKey for mutual TLS when talking to other orderers.
    ClientCertificate:
    ClientPrivateKey:
    # Separate intra-cluster listener (optional)
    ListenPort:
    ListenAddress:
    ServerCertificate:
    ServerPrivateKey:
    # ReplicationPolicy: ignored in etcdraft (always simple)
    ReplicationPolicy:
```

- **Khuyến nghị triển khai**:
  - Chọn số node lẻ để tối ưu chịu lỗi: 3 (chịu 1 lỗi), 5 (chịu 2 lỗi), 7 (chịu 3 lỗi).
  - Phân tán node qua nhiều DC/vùng; tránh đồng vị trí leader và đa số trên cùng một failure domain.
  - Bật TLS/mTLS giữa orderer và intra-cluster (`General.TLS`, `General.Cluster`).
  - Theo dõi dung lượng `WALDir/SnapDir`, đặt ngưỡng snapshot phù hợp thông qua cấu hình kênh (kích thước block, BatchSize/Timeout) và dung lượng đĩa.
  - Ứng dụng luôn đợi commit event từ peer, xử lý timeout và idempotency cho giao dịch gửi lại.

---

Ghi chú:
- Các trích dẫn đã rút gọn để minh hoạ nhanh. Xem thêm `orderer.yaml` và phần consensus `etcdraft` nếu cần tuỳ biến nâng cao.


