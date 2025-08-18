## Ledger

Audience: Architects, Application and smart contract developers, administrators

Một ledger trong Hyperledger Fabric lưu trữ thông tin sự kiện (facts) về các đối tượng nghiệp vụ: vừa giá trị hiện tại của thuộc tính (world state), vừa lịch sử giao dịch dẫn tới các giá trị hiện tại (blockchain).

### What is a Ledger?

Ledger thể hiện trạng thái hiện tại (ví dụ số dư) và chuỗi giao dịch theo thứ tự tạo ra trạng thái đó.

### Ledgers, Facts, and States

Ledger không lưu trực tiếp đối tượng mà lưu facts về trạng thái của đối tượng và lịch sử các thay đổi. Các ứng dụng tương tác qua smart contract dùng các API get/put/delete state.

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

Ledger gồm hai phần liên quan: world state (cơ sở dữ liệu giá trị hiện tại các key) và blockchain (nhật ký giao dịch bất biến, xác định world state).

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

World state lưu giá trị hiện tại của các key (key-value). Ứng dụng truy cập trực tiếp qua các API của smart contract. Phiên bản (version) nội bộ tăng dần để đảm bảo kiểm soát đồng thời.

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

### Blockchain

Blockchain là chuỗi các block liên kết bằng hash. Mỗi block chứa danh sách giao dịch và hash của block trước đó, tạo tính bất biến.

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

### Blocks

Block gồm: Header (Number, DataHash, PreviousHash), Data (danh sách giao dịch), Metadata (chữ ký, bitmap valid/invalid, vv.).

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

Một giao dịch gồm header, chữ ký, proposal input, response (RW-set) và các endorsements. RW-set mô tả các đọc/ghi mà smart contract thực hiện.

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

### World State database options

World state có thể dùng LevelDB (mặc định, nhúng) hoặc CouchDB (ngoài tiến trình, hỗ trợ JSON query). Cấu hình qua `StateDBConfig`.

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

Mỗi chaincode có namespace state riêng; blockchain không namespaced. Với private data, mỗi collection có không gian khoá riêng.

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

Mỗi channel có một ledger riêng (blockchain + world state). Khi dùng CouchDB, tên DB có chứa tên kênh để tách biệt.

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


