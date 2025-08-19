## Peers

Audience: Architects, Developers, Administrators

Tài liệu này tóm tắt vai trò peer trong Fabric và chèn các trích đoạn mã triển khai: peer lưu ledger/chaincode, Gateway service, và luồng xử lý giao dịch ở phía peer.

### Peers, ledgers, chaincodes

- Peer là nút lưu trữ bản sao ledger của từng kênh đã join và chạy chaincode container để thực thi smart contract.
- Một peer có thể tham gia nhiều kênh → giữ nhiều ledger và cài nhiều chaincode.

### Gateway service trên peer (từ v2.4)

- Peer nhúng dịch vụ Fabric Gateway để thay ứng dụng thực hiện proposal/endorsement/submit/wait.

Code minh hoạ (đăng ký Endorser và Gateway trên gRPC server của peer):

```895:915:internal/peer/node/start.go
auth := authHandler.ChainFilters(serverEndorser, authFilters...)
pb.RegisterEndorserServer(peerServer.Server(), auth)
if coreConfig.GatewayOptions.Enabled && coreConfig.DiscoveryEnabled {
    gatewayServer := gateway.CreateServer(auth, discoveryService, peerInstance, &serverConfig.SecOpts, aclProvider, coreConfig.LocalMSPID, coreConfig.GatewayOptions, builtinSCCs)
    gatewayprotos.RegisterGatewayServer(peerServer.Server(), gatewayServer)
}
```

Code minh hoạ (tạo Gateway server sử dụng Endorser và Ledger provider):

```56:87:internal/pkg/gateway/gateway.go
func CreateServer(localEndorser peerproto.EndorserServer, discovery Discovery, peerInstance *peer.Peer, secureOptions *comm.SecureOptions, policy ACLChecker, localMSPID string, options config.Options, systemChaincodes scc.BuiltinSCCs) *Server {
    adapter := &ledger.PeerAdapter{ Peer: peerInstance }
    notifier := commit.NewNotifier(adapter)
    server := newServer(&EndorserServerAdapter{ Server: localEndorser }, discovery, commit.NewFinder(adapter, notifier), policy, adapter, peerInstance.GossipService.SelfMembershipInfo(), localMSPID, secureOptions, options, systemChaincodes, peerInstance.OrdererEndpointOverrides, peerInstance.GetChannelConfig)
    peerInstance.AddConfigCallbacks(server.registry.configUpdate)
    return server
}
```

### Chu trình khởi tạo peer và hệ sinh thái xung quanh

- Peer khởi tạo LedgerMgr, Gossip service, Chaincode support, System chaincodes (QSCC, CSCC, _lifecycle_), Endorser, rồi mở gRPC server.

Code minh hoạ (rút gọn các bước chính):

```445:483:internal/peer/node/start.go
peerInstance.LedgerMgr = ledgermgmt.NewLedgerMgr(&ledgermgmt.Initializer{ /* StateListeners, Lifecycle, Metrics, LedgerConfig... */ })
peerServer, _ := comm.NewGRPCServer(listenAddr, serverConfig)
gossipService, _ := initGossipService(..., peerServer, signingIdentity, ...)
peerInstance.GossipService = gossipService
```

```737:767:internal/peer/node/start.go
endorserSupport := &endorser.SupportImpl{ SignerSerializer: signingIdentity, Peer: peerInstance, ChaincodeSupport: chaincodeSupport, ACLProvider: aclProvider, BuiltinSCCs: builtinSCCs }
serverEndorser := &endorser.Endorser{ PrivateDataDistributor: gossipService, ChannelFetcher: channelFetcher, LocalMSP: localMSP, Support: endorserSupport, Metrics: endorser.NewMetrics(metricsProvider) }
```

### Ứng dụng ↔ Peer: luồng xử lý giao dịch (tóm tắt)

1) Ứng dụng gửi proposal tới Gateway; Gateway chọn peer phù hợp, thực thi chaincode để tạo RW-set và thu thập endorsements theo policy.
2) Ứng dụng gửi envelope đã ký lại cho Gateway → chuyển tới orderer.
3) Khi block được phân phối về, peer validate (endorsement, MVCC/phantom), commit world state và phát commit event.

### Nhiều kênh và nhiều ledger

- Peer có thể host nhiều ledger (mỗi ledger tương ứng một kênh). Chaincode có thể được commit trên các kênh khác nhau, tuỳ chính sách.

---

Ghi chú:
- Các trích dẫn đã rút gọn để minh hoạ rõ vai trò peer và liên hệ tới mã nguồn khởi tạo/đăng ký dịch vụ.


