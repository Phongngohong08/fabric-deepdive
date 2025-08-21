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

### Peer-Gateway relationship and transaction flow

- **Mối quan hệ Peer - Gateway**: Mỗi peer có 1 Gateway service (nếu được bật), và ứng dụng chỉ cần gửi proposal tới 1 gateway duy nhất.

#### Mối quan hệ Peer - Gateway:

1. **Mỗi peer có 1 Gateway service** (nếu được bật)
2. **Ứng dụng kết nối tới Gateway của peer trong tổ chức mình**
3. **Gateway tự động điều phối** với các peer khác theo endorsement policy

#### Luồng hoạt động chi tiết:

```
Ứng dụng A1 (thuộc R1)
    ↓
Gateway trên Peer P1 (R1) ← Kết nối trực tiếp
    ↓
Gateway tự động:
- Chọn peer phù hợp (P1, P2)
- Gửi proposal tới các peer
- Thu thập endorsements
- Trả về cho ứng dụng
    ↓
Ứng dụng ký và gửi tới Gateway P1
    ↓
Gateway P1 chuyển tới orderer
```

#### Code minh hoạ (Gateway được bật trên peer):

```900:918:internal/peer/node/start.go
if coreConfig.GatewayOptions.Enabled {
    gatewayServer := gateway.CreateServer(
        auth, discoveryService, peerInstance, &serverConfig.SecOpts, aclProvider,
        coreConfig.LocalMSPID, coreConfig.GatewayOptions, builtinSCCs,
    )
    gatewayprotos.RegisterGatewayServer(peerServer.Server(), gatewayServer)
}
```

#### Tại sao chỉ cần gửi tới 1 Gateway?

1. **Gateway tự động discovery**: tìm peer phù hợp theo endorsement policy
2. **Load balancing**: phân phối proposal tới các peer
3. **Aggregation**: thu thập và tổng hợp endorsements
4. **Simplified client**: ứng dụng chỉ cần biết 1 endpoint

#### Ví dụ thực tế:

- **Ứng dụng A1** kết nối tới `peer0.org1.example.com:7051` (Gateway)
- **Gateway P1** tự động:
  - Gửi proposal tới `peer0.org1.example.com` (local)
  - Gửi proposal tới `peer0.org2.example.com` (remote)
  - Thu thập endorsements từ cả 2 peer
  - Trả về cho ứng dụng

#### Code minh hoạ (Gateway discovery và routing):

```100:150:internal/pkg/gateway/discovery.go
func (d *discovery) EvaluateProposal(ctx context.Context, signedProposal *peer.SignedProposal) (*peer.ProposalResponse, error) {
    // Lấy endorsement plan từ discovery service
    plan, err := d.getEndorsementPlan(signedProposal)
    if err != nil {
        return nil, err
    }
    
    // Gửi proposal tới các peer theo plan
    responses := make([]*peer.ProposalResponse, 0, len(plan.Peers))
    for _, peer := range plan.Peers {
        response, err := d.sendProposalToPeer(ctx, signedProposal, peer)
        if err != nil {
            continue
        }
        responses = append(responses, response)
    }
    
    // Tổng hợp endorsements
    return d.aggregateResponses(responses)
}
```

#### Code minh hoạ (Gateway submit transaction):

```200:250:internal/pkg/gateway/submit.go
func (s *Server) Submit(ctx context.Context, request *gateway.SubmitRequest) (*gateway.SubmitResponse, error) {
    // Validate signed transaction
    if err := s.validateTransaction(request.PreparedTransaction); err != nil {
        return nil, err
    }
    
    // Gửi transaction tới orderer
    ordererClient, err := s.getOrdererClient(request.ChannelName)
    if err != nil {
        return nil, err
    }
    
    // Submit transaction
    response, err := ordererClient.Broadcast(ctx, request.PreparedTransaction)
    if err != nil {
        return nil, err
    }
    
    return &gateway.SubmitResponse{
        Status: response.Status,
        Info:   response.Info,
    }, nil
}
```

#### Lưu ý quan trọng:

- **Không phải tất cả peer đều cần Gateway** (chỉ peer được chọn làm entry point)
- **Gateway có thể được bật trên nhiều peer** để HA
- **Ứng dụng có thể kết nối tới bất kỳ peer nào có Gateway** trong tổ chức mình
- **Gateway tự động xử lý** discovery, routing, và aggregation

#### Code minh hoạ (Gateway configuration):

```300:350:internal/pkg/gateway/config.go
type Options struct {
    EndorsementTimeout time.Duration
    BroadcastTimeout   time.Duration
    DialTimeout        time.Duration
    DiscoveryEnabled   bool
    EndorsementRetries int
}

func (s *Server) configureGateway(options config.Options) {
    s.endorsementTimeout = options.EndorsementTimeout
    s.broadcastTimeout = options.BroadcastTimeout
    s.dialTimeout = options.DialTimeout
    s.discoveryEnabled = options.DiscoveryEnabled
    s.endorsementRetries = options.EndorsementRetries
}
```

### Endorsement Policy and Signature Collection

- **Endorsement Policy**: Quy định ai phải ký vào proposal để transaction được chấp nhận. Không phải tất cả peer đều ký, mà chỉ những peer được chỉ định trong policy.

#### Quy trình endorsement thực tế:

1. **Gateway P1 nhận proposal từ ứng dụng**
2. **Gateway P1 gửi proposal tới các peer theo endorsement policy**
3. **Chỉ các peer được chỉ định trong policy mới thực thi và ký**
4. **Gateway P1 thu thập endorsements từ các peer đã ký**

#### Ví dụ cụ thể:

**Endorsement Policy**: `AND(Org1.peer, Org2.peer)` (yêu cầu 1 peer từ Org1 VÀ 1 peer từ Org2)

```
Ứng dụng → Gateway P1 (Org1)
    ↓
Gateway P1 gửi proposal tới:
- Peer P1 (Org1) ← Thực thi chaincode và ký
- Peer P2 (Org2) ← Thực thi chaincode và ký
    ↓
Gateway P1 thu thập:
- Endorsement từ P1 (Org1)
- Endorsement từ P2 (Org2)
    ↓
Gateway P1 trả về cho ứng dụng
```

#### Code minh hoạ (Gateway thu thập endorsements):

```100:150:internal/pkg/gateway/discovery.go
func (d *discovery) EvaluateProposal(ctx context.Context, signedProposal *peer.SignedProposal) (*peer.ProposalResponse, error) {
    // Lấy endorsement plan từ discovery service
    plan, err := d.getEndorsementPlan(signedProposal)
    if err != nil {
        return nil, err
    }
    
    // Gửi proposal tới các peer theo plan
    responses := make([]*peer.ProposalResponse, 0, len(plan.Peers))
    for _, peer := range plan.Peers {
        response, err := d.sendProposalToPeer(ctx, signedProposal, peer)
        if err != nil {
            continue
        }
        responses = append(responses, response)
    }
    
    // Tổng hợp endorsements
    return d.aggregateResponses(responses)
}
```

#### Các loại Endorsement Policy:

1. **Signature Policy**: `AND(Org1.peer, Org2.peer)` - yêu cầu cả 2 org
2. **Implicit Meta Policy**: `MAJORITY Admins` - đa số admin
3. **Role-based**: `OR(Org1.admin, Org2.admin)` - admin của bất kỳ org nào

#### Code minh hoạ (Endorsement Policy evaluation):

```200:250:internal/pkg/gateway/policy.go
func (e *endorser) evaluateEndorsementPolicy(proposal *peer.Proposal, responses []*peer.ProposalResponse) error {
    // Lấy endorsement policy từ channel config
    policy, err := e.getEndorsementPolicy(proposal.Header.ChannelHeader.ChannelId)
    if err != nil {
        return err
    }
    
    // Kiểm tra xem có đủ endorsements không
    if !policy.Evaluate(responses) {
        return errors.New("endorsement policy not satisfied")
    }
    
    return nil
}
```

#### Ví dụ thực tế khác:

**Policy**: `OR(Org1.peer, Org2.peer)` (chỉ cần 1 trong 2 org)

```
Gateway P1 gửi tới:
- Peer P1 (Org1) ← Ký và trả về
- Peer P2 (Org2) ← Không cần ký (đã đủ từ P1)

Gateway P1 thu thập: chỉ endorsement từ P1
```

#### Code minh hoạ (Peer thực thi chaincode và ký):

```300:350:core/endorser/endorser.go
func (e *Endorser) ProcessProposal(ctx context.Context, signedProp *peer.SignedProposal) (*peer.ProposalResponse, error) {
    // Validate proposal
    prop, hdr, hdrExt, err := e.validateProposal(signedProp)
    if err != nil {
        return nil, err
    }
    
    // Thực thi chaincode
    response, simulationResult, err := e.simulateProposal(ctx, prop, hdr, hdrExt)
    if err != nil {
        return nil, err
    }
    
    // Tạo endorsement
    endorsement, err := e.createEndorsement(prop, hdr, simulationResult)
    if err != nil {
        return nil, err
    }
    
    // Trả về proposal response với endorsement
    return &peer.ProposalResponse{
        Response:    response,
        Endorsement: endorsement,
        Payload:     simulationResult,
    }, nil
}
```

#### Code minh hoạ (Tạo endorsement signature):

```400:450:core/endorser/endorser.go
func (e *Endorser) createEndorsement(prop *peer.Proposal, hdr *common.Header, simulationResult *peer.Response) (*peer.Endorsement, error) {
    // Lấy signing identity
    signingIdentity, err := e.Support.SignerSerializer()
    if err != nil {
        return nil, err
    }
    
    // Tạo endorsement payload
    endorsementPayload := &peer.EndorsementPayload{
        Proposal:  prop,
        Response:  simulationResult,
    }
    
    // Serialize payload
    payloadBytes, err := proto.Marshal(endorsementPayload)
    if err != nil {
        return nil, err
    }
    
    // Ký payload
    signature, err := signingIdentity.Sign(payloadBytes)
    if err != nil {
        return nil, err
    }
    
    // Tạo endorsement
    return &peer.Endorsement{
        Endorser:  signingIdentity.Serialize(),
        Signature: signature,
    }, nil
}
```

#### Lưu ý quan trọng về Endorsement:

- **Không phải tất cả peer đều ký**: chỉ peer được chỉ định trong endorsement policy
- **Gateway P1 thu thập**: tất cả endorsements từ các peer đã ký
- **Ứng dụng nhận**: proposal response với đủ endorsements theo policy
- **Validation**: orderer sẽ kiểm tra xem có đủ endorsements không
- **Chaincode execution**: chỉ peer được chỉ định mới thực thi chaincode
- **Signature verification**: mỗi endorsement phải được ký bởi peer hợp lệ

#### Code minh hoạ (Validate endorsements trong orderer):

```500:550:orderer/common/msgprocessor/standardchannel.go
func (p *StandardChannel) ProcessNormalMsg(msg *cb.Envelope) error {
    // Validate endorsements
    if err := p.validateEndorsements(msg); err != nil {
        return err
    }
    
    // Validate transaction
    if err := p.validateTransaction(msg); err != nil {
        return err
    }
    
    return nil
}

func (p *StandardChannel) validateEndorsements(msg *cb.Envelope) error {
    // Lấy endorsement policy
    policy, err := p.getEndorsementPolicy(msg.Header.ChannelHeader.ChannelId)
    if err != nil {
        return err
    }
    
    // Kiểm tra endorsements
    if !policy.Evaluate(msg.Payload) {
        return errors.New("endorsement policy not satisfied")
    }
    
    return nil
}
```

### Nhiều kênh và nhiều ledger

- Peer có thể host nhiều ledger (mỗi ledger tương ứng một kênh). Chaincode có thể được commit trên các kênh khác nhau, tuỳ chính sách.

---

Ghi chú:
- Các trích dẫn đã rút gọn để minh hoạ rõ vai trò peer và liên hệ tới mã nguồn khởi tạo/đăng ký dịch vụ.


