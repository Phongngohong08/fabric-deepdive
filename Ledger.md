## Ledger

Audience: Architects, Application and smart contract developers, administrators

Một ledger trong Hyperledger Fabric lưu trữ thông tin sự kiện (facts) về các đối tượng nghiệp vụ: vừa giá trị hiện tại của thuộc tính (world state), vừa lịch sử giao dịch dẫn tới các giá trị hiện tại (blockchain).

### What is a Ledger?

Ledger là sổ ghi chép số hoá của doanh nghiệp:
- Trạng thái hiện tại (world state) của các đối tượng nghiệp vụ dưới dạng cặp key–value.
- Dòng thời gian bất biến các giao dịch (blockchain) tạo ra trạng thái hiện tại.

Bạn có thể liên tưởng tới tài khoản ngân hàng: số dư là trạng thái hiện tại; còn sao kê là chuỗi giao dịch dẫn tới số dư đó.

### Ledgers, Facts, and States

- Ledger lưu "facts" (sự kiện/thuộc tính) về đối tượng, không nhất thiết bản thân đối tượng.
- Mỗi đối tượng ứng với một hoặc nhiều state (key–value) trong world state.
- Lịch sử facts là bất biến: chỉ có thể thêm mới (append) qua các giao dịch; không sửa xoá quá khứ.
- Ứng dụng tương tác qua smart contract bằng các API đọc/ghi/xoá state.

Code minh hoạ (các API state trong Chaincode Shim):

```go
// vendor/github.com/hyperledger/fabric-chaincode-go/v2/shim/interfaces.go (trích)
// GetState/PutState/DelState là các API truy cập world state ở cấp chaincode
GetState(key string) ([]byte, error)
GetMultipleStates(keys ...string) ([][]byte, error)
PutState(key string, value []byte) error
DelState(key string) error
```

Code minh hoạ (ví dụ asset-transfer-basic sử dụng PutState/GetState/DelState):

```go
// fabric-samples/asset-transfer-basic/chaincode-go/chaincode/smartcontract.go (trích)
func (s *SmartContract) InitLedger(ctx contractapi.TransactionContextInterface) error {
    assets := []Asset{
        {ID: "asset1", Color: "blue", Size: 5, Owner: "Tomoko", AppraisedValue: 300},
        {ID: "asset2", Color: "red", Size: 5, Owner: "Brad", AppraisedValue: 400},
        // ...
    }
    for _, asset := range assets {
        assetJSON, _ := json.Marshal(asset)
        if err := ctx.GetStub().PutState(asset.ID, assetJSON); err != nil { return err }
    }
    return nil
}

func (s *SmartContract) ReadAsset(ctx contractapi.TransactionContextInterface, id string) (*Asset, error) {
    assetJSON, err := ctx.GetStub().GetState(id)
    if err != nil || assetJSON == nil { return nil, fmt.Errorf("not found") }
    var asset Asset
    _ = json.Unmarshal(assetJSON, &asset)
    return &asset, nil
}

func (s *SmartContract) DeleteAsset(ctx contractapi.TransactionContextInterface, id string) error {
    return ctx.GetStub().DelState(id)
}
```

### The Ledger

Ledger có hai phần liên hệ chặt chẽ:
- World state: cơ sở dữ liệu các giá trị hiện tại theo key, tối ưu cho truy vấn trực tiếp.
- Blockchain: nhật ký giao dịch theo block, bất biến, là nguồn gốc để suy ra world state.

Code minh hoạ (struct `cb.Block` tối giản):

```go
// vendor/github.com/hyperledger/fabric-protos-go-apiv2/common/common.pb.go (trích)
type Block struct {
    Header   *BlockHeader
    Data     *BlockData
    Metadata *BlockMetadata
}
```

Code minh hoạ (tạo block và header cơ bản):

```go
// protoutil/blockutils.go (trích)
func NewBlock(seqNum uint64, previousHash []byte) *cb.Block {
    block := &cb.Block{}
    block.Header = &cb.BlockHeader{ Number: seqNum, PreviousHash: previousHash, DataHash: []byte{} }
    block.Data = &cb.BlockData{}
    // Khởi tạo metadata rỗng cho các chỉ mục
    var metadataContents [][]byte
    for i := 0; i < len(cb.BlockMetadataIndex_name); i++ { metadataContents = append(metadataContents, []byte{}) }
    block.Metadata = &cb.BlockMetadata{Metadata: metadataContents}
    return block
}
```

### World State

- Lưu giá trị hiện tại của các key (key–value) cho mỗi namespace chaincode.
- Hỗ trợ truy vấn theo range, composite key, hoặc truy vấn JSON khi dùng CouchDB.
- Mỗi key có version tăng dần; khi commit sẽ kiểm tra xung đột dựa trên version (MVCC), đảm bảo tính nhất quán.

Code minh hoạ (truy vấn theo range trong namespace chaincode):

```go
// fabric-samples/asset-transfer-basic/chaincode-go/chaincode/smartcontract.go (trích)
func (s *SmartContract) GetAllAssets(ctx contractapi.TransactionContextInterface) ([]*Asset, error) {
    it, err := ctx.GetStub().GetStateByRange("", "")
    if err != nil { return nil, err }
    defer it.Close()
    var assets []*Asset
    for it.HasNext() {
        resp, _ := it.Next()
        var a Asset
        _ = json.Unmarshal(resp.Value, &a)
        assets = append(assets, &a)
    }
    return assets, nil
}
```

Giải thích ngắn về MVCC (Multi-Version Concurrency Control):
- Khi smart contract được endorse, hệ thống ghi lại Read-Write set (RW-set), trong đó Read set chứa cặp (key, version) của tất cả các key đã đọc.
- Khi commit block, peer sẽ kiểm tra version hiện tại của mỗi key khớp với version trong Read set. Nếu có giao dịch khác đã cập nhật key đó sau thời điểm endorse, version bị lệch và giao dịch sẽ bị đánh dấu không hợp lệ (MVCC read conflict) và không cập nhật world state.
- Với truy vấn theo range hoặc truy vấn JSON (CouchDB), Fabric thêm thông tin phạm vi truy vấn và/hoặc hash kết quả vào Read set để phát hiện phantom reads. Nếu có key được thêm/xoá/đổi trong phạm vi truy vấn giữa lúc endorse và commit, giao dịch cũng sẽ bị vô hiệu.
- Ưu điểm: song song hoá cao, tránh khoá dài; Nhược điểm: có thể bị xung đột và phải gửi lại. Thực tiễn: chia nhỏ giao dịch, dùng kiểm tra điều kiện (check-and-set) trên giá trị/phiên bản, thiết kế khoá logic, và tận dụng chỉ mục/truy vấn phù hợp khi dùng CouchDB.

### Blockchain

- Chuỗi các block liên kết bằng hash của header, đảm bảo tính bất biến và phát hiện chỉnh sửa.
- Mỗi block chứa danh sách giao dịch (block data) và metadata (chữ ký người tạo block, bitmap kết quả validate, ...).
- Dịch vụ ordering chịu trách nhiệm sắp xếp và cắt block theo thứ tự.

Code minh hoạ (tính hash header và dữ liệu block):

```go
// protoutil/blockutils.go (trích)
func BlockHeaderHash(b *cb.BlockHeader) []byte {
    sum := sha256.Sum256(BlockHeaderBytes(b))
    return sum[:]
}

func ComputeBlockDataHash(b *cb.BlockData) []byte {
    sum := sha256.Sum256(bytes.Join(b.Data, nil))
    return sum[:]
}
```

Code minh hoạ (các struct Block/BlockHeader/BlockData/BlockMetadata ở protobuf Go):

```go
// vendor/github.com/hyperledger/fabric-protos-go-apiv2/common/common.pb.go (trích)
type Block struct {
    Header   *BlockHeader
    Data     *BlockData
    Metadata *BlockMetadata
}

type BlockHeader struct {
    Number       uint64 // vị trí trong chuỗi khối
    PreviousHash []byte // hash header của block trước
    DataHash     []byte // hash của BlockData
}

type BlockData struct {
    Data [][]byte // danh sách envelope (giao dịch) dạng bytes
}

type BlockMetadata struct {
    Metadata [][]byte // các metadata: chữ ký, bitmap valid/invalid, v.v.
}
```

Giải thích thêm (tính bất biến và toàn vẹn trong Blockchain):
- Header của mỗi block chứa `DataHash` (hash của toàn bộ phần Data) và `PreviousHash` (hash header block trước). Khi bất kỳ byte nào trong Data bị thay đổi, `DataHash` khác đi, kéo theo hash header khác, làm đứt chuỗi hash về sau. Vì vậy sổ cái là bất biến theo nghĩa "tamper-evident" (mọi sửa đổi đều bị phát hiện).
- `BlockMetadata` không tham gia vào hash header nhằm tách biệt dữ liệu giao dịch (được băm và xâu chuỗi) với siêu dữ liệu thêm sau (chữ ký người tạo block, bitmap hợp lệ/không hợp lệ, v.v.). Siêu dữ liệu vẫn được bảo vệ bằng chữ ký và quy trình xác thực khi commit.
- Dịch vụ ordering sắp xếp giao dịch, cắt block, và thường ký khối (tuỳ cơ chế). Peer khi nhận block sẽ: (1) kiểm tra chữ ký/siêu dữ liệu, (2) xác minh hash header (bao gồm `PreviousHash`), (3) validate từng giao dịch, (4) commit block kèm cờ valid/invalid vào metadata.
- Hệ quả bảo mật: kẻ tấn công không thể đơn phương sửa block ở một peer rồi thuyết phục mạng; vì chuỗi hash và chữ ký/đồng thuận đảm bảo tất cả nút phải nhất quán thứ tự và nội dung khối.

### Blocks

Một block có 3 phần:
- Header: Number, DataHash (hash của phần Data), PreviousHash (hash header block trước).
- Data: danh sách các giao dịch (mảng các envelope bytes).
- Metadata: thông tin chữ ký người tạo block, bitmap valid/invalid của từng giao dịch, hash tích luỹ trạng thái để phát hiện fork, v.v.

Code minh hoạ (tuần tự hoá thành phần block):

```go
// common/ledger/blkstorage/block_serialization.go (trích)
type serializedBlockInfo struct {
    blockHeader *common.BlockHeader
    txOffsets   []*txindexInfo
    metadata    *common.BlockMetadata
}

func addHeaderBytes(h *common.BlockHeader, buf []byte) []byte {
    buf = protowire.AppendVarint(buf, h.Number)
    buf = protowire.AppendBytes(buf, h.DataHash)
    buf = protowire.AppendBytes(buf, h.PreviousHash)
    return buf
}
```

### Transactions

Một giao dịch endorser trong Fabric gồm các phần chính:
- Header: metadata như loại giao dịch, channel, TxID, timestamp, epoch.
- Signature: chữ ký số của client, chống giả mạo.
- Proposal input (chaincode invocation spec): tham số đầu vào cho smart contract.
- Response: kết quả thực thi smart contract chứa Read-Write set (RW-set).
- Endorsements: tập chữ ký từ các tổ chức theo chính sách endorsement; chỉ khi đủ chữ ký thì giao dịch mới được coi là hợp lệ để cập nhật world state.

Code minh hoạ (cấu trúc dùng trong validate, bao gồm RW-set):

```go
// core/ledger/kvledger/txmgmt/validation/types.go (trích)
type transaction struct {
    indexInBlock            int
    id                      string
    rwset                   *rwsetutil.TxRwSet
    validationCode          peer.TxValidationCode
    containsPostOrderWrites bool
}
```

Code minh hoạ (các struct bao bọc giao dịch ở lớp protobuf: Payload/Envelope/ChannelHeader/SignatureHeader):

```go
// vendor/github.com/hyperledger/fabric-protos-go-apiv2/common/common.pb.go (trích)
type Payload struct {
    Header *Header // gồm ChannelHeader và SignatureHeader (dạng bytes)
    Data   []byte  // nội dung theo loại trong header (ví dụ EndorserTransaction)
}

type Envelope struct {
    Payload   []byte // marshaled Payload
    Signature []byte // chữ ký của creator trong Header
}

type ChannelHeader struct {
    Type        int32
    Version     int32
    Timestamp   *timestamppb.Timestamp
    ChannelId   string
    TxId        string
    Epoch       uint64
    Extension   []byte
    TlsCertHash []byte
}

type SignatureHeader struct {
    Creator []byte // MSP SerializedIdentity
    Nonce   []byte // chống replay
}
```

### World State database options

- LevelDB: mặc định, nhúng trong tiến trình peer, phù hợp key–value đơn giản, hiệu năng cao.
- CouchDB: tiến trình riêng, phù hợp dữ liệu JSON và truy vấn phong phú (selector, index); vẫn trong suốt với smart contract.
- Fabric "pluggable": có thể thay thế backend trong tương lai (quan hệ, đồ thị, thời gian, ...).

Code minh hoạ (cấu hình StateDB và CouchDB):

```go
// core/ledger/ledger_interface.go (trích)
type StateDBConfig struct {
    StateDatabase string   // "goleveldb" hoặc "CouchDB"
    CouchDB *CouchDBConfig // dùng khi StateDatabase = "CouchDB"
}

type CouchDBConfig struct {
    Address string
    Username string
    Password string
    MaxRetries int
    MaxRetriesOnStartup int
    RequestTimeout time.Duration
    InternalQueryLimit int
    MaxBatchUpdateSize int
    CreateGlobalChangesDB bool
    RedoLogPath string
    UserCacheSizeMBs int
}
```

### Example Ledger: Basic Asset Transfer

Ví dụ khởi tạo 6 tài sản và truy vấn toàn bộ tài sản trong namespace chaincode.

Code minh hoạ (InitLedger và GetAllAssets):

```go
// fabric-samples/asset-transfer-basic/chaincode-go/chaincode/smartcontract.go (trích)
func (s *SmartContract) InitLedger(ctx contractapi.TransactionContextInterface) error {
    assets := []Asset{ /* ... 6 assets ... */ }
    for _, asset := range assets {
        assetJSON, _ := json.Marshal(asset)
        if err := ctx.GetStub().PutState(asset.ID, assetJSON); err != nil { return err }
    }
    return nil
}

func (s *SmartContract) GetAllAssets(ctx contractapi.TransactionContextInterface) ([]*Asset, error) {
    it, err := ctx.GetStub().GetStateByRange("", "")
    if err != nil { return nil, err }
    defer it.Close()
    var assets []*Asset
    for it.HasNext() {
        resp, _ := it.Next()
        var a Asset
        _ = json.Unmarshal(resp.Value, &a)
        assets = append(assets, &a)
    }
    return assets, nil
}
```

### Namespaces

- Mỗi chaincode có namespace riêng trong world state, cô lập dữ liệu giữa các chaincode.
- Blockchain không namespaced: một block có thể chứa giao dịch từ nhiều namespace khác nhau.
- Private data: mỗi collection là một không gian khoá tách biệt (với hậu tố $$p/$$h khi dùng CouchDB) cùng chính sách truy cập riêng.

Code minh hoạ (bố cục key trong lifecycle và collection key):

```go
// core/chaincode/lifecycle/lifecycle.go (trích – bố cục DB theo namespace)
// namespaces/metadata/<namespace> -> namespace metadata, including namespace type
// namespaces/fields/<namespace>/Sequence -> sequence for this namespace
// namespaces/fields/<namespace>/<field> -> field of namespace type

// Private/Org Scope Implicit Collection layout
// namespaces/metadata/<namespace>#<sequence_number> -> namespace metadata
// namespaces/fields/<namespace>#<sequence_number>/<field> -> field of namespace type
```

```go
// core/common/privdata/collection.go (trích)
const (
    collectionSeparator = "~"
    collectionSuffix = "collection"
)
func BuildCollectionKVSKey(ccname string) string { return ccname + collectionSeparator + collectionSuffix }
```

### Channels

- Mỗi channel có ledger riêng hoàn toàn (blockchain + world state + namespaces).
- Có thể trao đổi thông tin giữa các channel qua chaincode-to-chaincode hoặc ứng dụng, nhưng commit và validate vẫn độc lập.

Code minh hoạ (đặt tên DB CouchDB theo chain/channel):

```go
// core/ledger/kvledger/txmgmt/statedb/statecouchdb/couchdbutil.go (trích)
// constructMetadataDBName: dbName = <channelName hoặc truncated> + "_"
// constructNamespaceDBName: namespace định dạng
//   <chaincode>                 (public data)
//   <chaincode>$$p<collection>  (private data)
//   <chaincode>$$h<collection>  (hashes of private data)
```

---

Ghi chú: Các đoạn mã đã được trích gọn để minh hoạ trực tiếp cho từng mục lý thuyết. Tham khảo tệp nguồn đầy đủ trong repo để xem chi tiết triển khai.


