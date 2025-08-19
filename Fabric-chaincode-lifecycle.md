## Fabric chaincode lifecycle

Audience: Operators, Administrators, Architects

Tài liệu này tóm tắt quy trình lifecycle (package → install → approve → commit) và trích mã CLI/peer phục vụ triển khai.

### Tổng quan

- Lifecycle cho phép các tổ chức đồng thuận trước về định nghĩa chaincode (tên, phiên bản, sequence, endorsement policy, collections, init‑required, plugin…)
- Bật bằng channel capability `V2_0`. Lifecycle mặc định dùng policy `Channel/Application/LifecycleEndorsement` (đa số org) cho bước approve/commit định nghĩa.

### Các bước chính

1) Package chaincode → file `.tar.gz` gồm `metadata.json` và `code.tar.gz`.
2) Install package lên các peer của mỗi tổ chức sẽ endorse/query.
3) Approve chaincode definition cho từng tổ chức (bao gồm `name`, `version`, `sequence`, `packageID`, policy/collections…)
4) Commit chaincode definition lên kênh khi đủ tổ chức đã approve; container chaincode được khởi chạy khi invoke đầu tiên (hoặc Init nếu `init-required`).

### Lệnh CLI (peer lifecycle)

Code minh hoạ (khai báo các subcommands):

```32:46:internal/peer/lifecycle/chaincode/chaincode.go
chaincodeCmd.AddCommand(PackageCmd(nil))
chaincodeCmd.AddCommand(CalculatePackageIDCmd(nil))
chaincodeCmd.AddCommand(InstallCmd(nil, cryptoProvider))
chaincodeCmd.AddCommand(ApproveForMyOrgCmd(nil, cryptoProvider))
chaincodeCmd.AddCommand(CheckCommitReadinessCmd(nil, cryptoProvider))
chaincodeCmd.AddCommand(CommitCmd(nil, cryptoProvider))
```

Code minh hoạ (Approve for my org – trích xử lý chính):

```126:208:internal/peer/lifecycle/chaincode/approveformyorg.go
// Tạo proposal, ký, gửi tới endorser(s), lắp ghép signed tx
proposal, txID, _ := a.createProposal(a.Input.TxID)
signedProposal, _ := signProposal(proposal, a.Signer)
for _, e := range a.EndorserClients { responses = append(responses, e.ProcessProposal(ctx, signedProposal)) }
env, _ := protoutil.CreateSignedTx(proposal, a.Signer, responses...)
// Broadcast & chờ event commit (nếu yêu cầu)
```

### Tự động hoá triển khai (mẫu trong test framework)

```63:92:integration/nwo/deploy.go
func DeployChaincode(...) {
  PackageAndInstallChaincode(...)
  ApproveChaincodeForMyOrg(...)
  CheckCommitReadinessUntilReady(...)
  CommitChaincode(...)
  if chaincode.InitRequired { InitChaincode(...) }
}
```

### Các trường trong chaincode definition (tóm tắt ý nghĩa)

- `Name`: tên gọi khi ứng dụng invoke.
- `Version`: nhãn logic do các bên thoả thuận để đồng bộ binary; peer không kiểm nội dung chuỗi.
- `Sequence`: số lần định nghĩa được commit trên kênh; tăng +1 cho mỗi lần cập nhật.
- `Endorsement Policy`: chữ ký tổ chức cần cho giao dịch hợp lệ; mặc định tham chiếu `Application/Endorsement`.
- `Collections`: tệp định nghĩa PDC nếu dùng private data.
- `InitRequired`: yêu cầu gọi `Init` trước khi invoke các hàm khác (khi dùng shim low‑level). Khuyến nghị nhúng init‑logic vào hàm nghiệp vụ thay vì bắt buộc `Init`.
- `ESCC/VSCC Plugins`: tuỳ chọn plugin tuỳ biến.
- `PackageID`: liên kết định nghĩa với gói đã cài trên peer của từng org.

### Nâng cấp (upgrade)

- Dùng lại quy trình: khi nâng cấp binary → đổi `Version` và `PackageID`, tăng `Sequence`. Khi chỉ đổi policy/collections → chỉ cần tăng `Sequence`.

### Tình huống điển hình

- Org mới tham gia kênh: cài package và approve định nghĩa hiện có → dùng được chaincode mà không cần commit lại.
- Cập nhật endorsement policy: approve definition mới với policy mới, tăng sequence, commit; không cần Init lại.
- Org không đồng ý định nghĩa: org đó không thể invoke/endorse; các org khác vẫn dùng bình thường nếu commit thành công.
- Khác `PackageID` giữa org: vẫn hợp lệ nếu tuân thủ cùng định nghĩa/policy – hỗ trợ logic tuỳ biến per‑org.

---

Ghi chú:
- Khi commit định nghĩa, cần target đủ peers thoả mãn `LifecycleEndorsement` policy; policy này độc lập với policy endorse giao dịch của chaincode.


