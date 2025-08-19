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
3) Peers validate và commit (xem tài liệu “Peers”).

### Raft (khuyến nghị)

- Mỗi kênh vận hành một cụm Raft riêng (consenter set). Một leader được bầu động, replicate log entries tới followers; khi đạt quorum → commit.
- Ưu điểm so với Kafka: cấu hình/triển khai đơn giản hơn, dễ phân tán nhiều tổ chức, được hỗ trợ trong cộng đồng Fabric.

---

Ghi chú:
- Các trích dẫn đã rút gọn để minh hoạ nhanh. Xem thêm `orderer.yaml` và phần consensus `etcdraft` nếu cần tuỳ biến nâng cao.


