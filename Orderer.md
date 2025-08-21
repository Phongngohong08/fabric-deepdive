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

### Raft (khuyến nghị)

- Mỗi kênh vận hành một cụm Raft riêng (consenter set). Một leader được bầu động, replicate log entries tới followers; khi đạt quorum → commit.
- Ưu điểm so với Kafka: cấu hình/triển khai đơn giản hơn, dễ phân tán nhiều tổ chức, được hỗ trợ trong cộng đồng Fabric.

---

Ghi chú:
- Các trích dẫn đã rút gọn để minh hoạ nhanh. Xem thêm `orderer.yaml` và phần consensus `etcdraft` nếu cần tuỳ biến nâng cao.


