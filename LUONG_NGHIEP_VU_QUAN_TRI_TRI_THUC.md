# Luồng nghiệp vụ cơ bản – Hệ thống Quản trị tri thức (KMS)

> Tài liệu này được tổng hợp từ code hiện tại của `kms-client` (Angular) và `kms-service` (.NET), cập nhật ngày 21/10/2026. Mục tiêu là để người dùng nghiệp vụ, BA, developer và tester cùng hiểu một luồng theo cùng một cách.

## 1. Cách đọc tài liệu

- **Người dùng**: đọc phần “Cách thao tác” và “Kết quả”.
- **BA**: đọc thêm “Quy tắc/điều kiện” và sơ đồ trạng thái.
- **Developer**: đối chiếu cột quyền, API và phần “Lưu ý theo code hiện tại”.
- **Tester**: dùng các điều kiện trước/sau và checklist cuối tài liệu làm test case cơ bản.
- Tên quyền trong ngoặc, ví dụ `DOC:VIEW`, là mã quyền RBAC trong hệ thống.
- “Xóa” là loại bỏ bản ghi; “Ngừng áp dụng” là giữ bản ghi nhưng không cho dùng tiếp; “Lưu trữ” là kết thúc vòng đời tài liệu. Ba thao tác này không đồng nghĩa.

## 2. Vai trò và nguyên tắc chung

| Nhóm vai trò | Trách nhiệm chính |
| --- | --- |
| Quản trị hệ thống | Quản lý người dùng, nhóm tài khoản, quyền, phạm vi phòng ban/danh mục |
| Quản trị tri thức | Quản lý danh mục, không gian, kho tài liệu, tài liệu hết hạn |
| Người đóng góp | Tải lên, phân loại, tạo/cập nhật tài liệu, gửi duyệt |
| Chuyên gia/người phê duyệt | Xem hàng chờ, phê duyệt, từ chối hoặc yêu cầu chỉnh sửa |
| Người dùng thông thường | Xem/tải/chia sẻ/đánh giá/góp ý tài liệu trong phạm vi được phép |
| Người xử lý góp ý | Tiếp nhận, xác nhận đã xử lý hoặc từ chối góp ý |

Quyền hiển thị menu chỉ là lớp đầu tiên. Backend vẫn kiểm tra quyền và phạm vi dữ liệu. Với tài liệu/không gian riêng tư, có quyền chức năng chưa chắc đã đủ; người dùng còn phải là chủ sở hữu, thành viên, người được chia sẻ hoặc thuộc phạm vi tổ chức/danh mục hợp lệ.

## 3. Luồng tổng thể của một tài liệu

```text
Nguồn tài liệu
  ├─ Kho tài liệu: tạo + tải file
  ├─ Không gian tri thức: tạo + tải file trong không gian
  ├─ Thu thập tri thức: tải file ngoài → phân loại
  └─ Tri thức chuyên gia nghỉ hưu: tạo hồ sơ + tải file
          │
          ▼
Tạo Document + DocumentVersion
          │
          ├─ Tài liệu trong Không gian → trạng thái Space, phiên bản được duyệt ngay
          │
          └─ Tài liệu thông thường → gửi workflow → Pending
                                             │
                           ┌─────────────────┼──────────────────┐
                           ▼                 ▼                  ▼
                        Approve           Reject          Request change
                           │                 │                  │
                bước tiếp theo/final     Rejected             Draft
                           │
                           ▼
                        Approved
                           │
                 đến/sau ngày hết hạn
                           ▼
                         Expired
                      ┌────┴────┐
                      ▼         ▼
                   Gia hạn    Lưu trữ
                   Approved   Archived
```

### 3.1. Trạng thái tài liệu

| Trạng thái kỹ thuật | Hiển thị/nghĩa nghiệp vụ | Hành động tiếp theo thường gặp |
| --- | --- | --- |
| `Draft` | Nháp/yêu cầu sửa | Chỉnh sửa, tạo phiên bản và gửi duyệt lại |
| `WaitingToSendCollect` | Chờ gửi duyệt của luồng thu thập | Bấm **Gửi duyệt** |
| `Pending` | Chờ duyệt | Người phê duyệt xử lý ở màn Phê duyệt tài liệu |
| `Approved` | Đã duyệt/đang hiệu lực | Xem, tải, chia sẻ, đánh giá, góp ý |
| `Rejected` | Từ chối | Chủ tài liệu chỉnh sửa và gửi lại bằng phiên bản mới |
| `Expired` | Hết hiệu lực | Gia hạn hoặc lưu trữ |
| `Archived` | Lưu trữ | Không được gia hạn trong UI hiện tại |
| `Space` | Tài liệu trong không gian | Được dùng trong không gian, không qua workflow thường |

### 3.2. Trạng thái phê duyệt

```text
Draft ──gửi duyệt──> Pending
Pending ──duyệt bước chưa cuối──> Pending ở bước kế tiếp
Pending ──duyệt bước cuối──> Approved
Pending ──từ chối──> Rejected
Pending ──yêu cầu sửa──> Draft
```

- Mỗi lần gửi duyệt gắn với **một phiên bản tài liệu**.
- Người duyệt được xác định động theo cấu hình workflow, danh mục, chuyên gia phụ trách, phòng ban/chức danh và quyền RBAC.
- Code runtime hiện chỉ hỗ trợ chế độ một người duyệt bất kỳ (`AnyOne`) ở mỗi bước.

## 4. Báo cáo & phân tích

**Mục đích:** xem tổng quan hoạt động tri thức, sức khỏe tài liệu và bảng xếp hạng người dùng/phòng ban.

**Quyền:** `DAS:VIEW` ở backend và menu.

### Cách thao tác

1. Vào **Báo cáo và phân tích**.
2. Chọn người dùng, phòng ban, từ ngày, đến ngày nếu cần.
3. Bấm **Áp dụng** để tải lại dữ liệu; bấm **Đặt lại** để bỏ điều kiện.
4. Tab **Tổng quan** hiển thị KPI, phân bố danh mục/phòng ban, cảnh báo và tình trạng tài liệu.
5. Tab **Bảng xếp hạng**:
   - Xem theo người dùng hoặc phòng ban.
   - Chọn tiêu chí: tổng điểm, đóng góp, làm sạch, phê duyệt.
   - Bấm một phòng ban để drill-down danh sách tài khoản trong phòng ban.
   - Lọc người đã/chưa đóng góp.
6. Chọn **Xuất Excel theo Phòng ban** hoặc **Xuất Excel theo Tài khoản**.

### Quy tắc và kết quả

- Đây là màn chỉ đọc; không có Thêm/Sửa/Xóa.
- Bộ lọc đang áp dụng được gửi cùng yêu cầu lấy dữ liệu và xuất Excel.
- Khoảng ngày phải hợp lệ về nghiệp vụ: từ ngày không lớn hơn đến ngày.
- API chính: `Reports/Overview`, `FilterOptions`, `ActiveUserRanking`, `DepartmentRanking`, `ExportDepartmentRankingExcel`, `ExportUserRankingExcel`.

## 5. Phân quyền

**Mục đích:** quản lý nhóm tài khoản, thành viên nhóm, quyền chức năng và quyền xem tài liệu theo danh mục/phòng ban.

**Quyền:** xem `ROLE:VIEW`; thêm/sửa/xóa `ROLE:CREATE`, `ROLE:UPDATE`, `ROLE:DELETE`; gán quyền `ROLE:ASSIGN_PERMISSIONS`; quản trị hệ thống có thể vào bằng `SYS:ADMIN`.

### 5.1. Nhóm tài khoản

**Thêm nhóm**

1. Chọn tab **Nhóm tài khoản** → **Thêm mới**.
2. Nhập mã nhóm, tên nhóm, mô tả, trạng thái hoạt động.
3. Lưu.
4. Kết quả: nhóm được tạo; mã nhóm phải duy nhất.

**Sửa/Ngừng áp dụng/Áp dụng lại**

1. Chọn **Sửa** tại nhóm.
2. Mã nhóm bị khóa, chỉ sửa tên, mô tả và trạng thái.
3. Bỏ/bật `isActive` tương ứng ngừng/kích hoạt lại nhóm.
4. Lưu và tải lại danh sách.

**Gán người dùng vào nhóm**

1. Chọn **Gán User vào nhóm**.
2. Tìm kiếm, chọn/bỏ chọn người dùng; có thể chọn tất cả danh sách đang lọc.
3. Lưu.
4. Hệ thống so sánh danh sách cũ/mới: thêm người mới chọn và gỡ người bị bỏ chọn.

**Xóa nhóm**

1. Chọn nhóm → **Xóa** → xác nhận.
2. Chỉ xóa được khi nhóm không còn thành viên và không còn role binding.
3. Nếu còn liên kết, backend trả conflict; cần gỡ thành viên và quyền trước.

### 5.2. Phân quyền nhóm tài khoản

1. Chọn tab **Phân quyền nhóm tài khoản**.
2. Chọn một nhóm.
3. Hệ thống tải catalog quyền đã được cấp, nhóm theo module/vai trò.
4. Chọn từng quyền hoặc chọn tất cả quyền của module.
5. Bấm **Lưu**.
6. Kết quả: các binding quyền phạm vi toàn hệ thống (`Global`) của nhóm được đồng bộ.

### 5.3. Phân quyền tài liệu theo danh mục/phòng ban

1. Chọn tab **Phân quyền tài liệu**.
2. Chọn danh mục tri thức.
3. Cây tổ chức hiển thị các phòng ban đã được phép.
4. Chọn một nút cha sẽ chọn cả nhánh con; bỏ chọn sẽ bỏ cả nhánh.
5. Bấm **Lưu phân quyền**.
6. Hệ thống chuẩn hóa nút cha khi toàn bộ con đã chọn và thay thế toàn bộ danh sách quyền cũ của danh mục.

**Điểm cần test:** tài khoản ngoài phòng ban được cấp không được nhìn thấy tài liệu thuộc danh mục đó; quyền trên danh mục và quyền chức năng tài liệu phải cùng thỏa mãn.

## 6. Quản lý người dùng

**Mục đích:** quản lý hồ sơ, tài khoản đăng nhập, nhóm tài khoản, trạng thái và mật khẩu.

**Quyền:** `SYS:ADMIN` cho toàn bộ controller và route.

### Danh sách và tìm kiếm

- Tìm theo họ tên, username, email.
- Lọc theo phòng ban, trạng thái và nhóm tài khoản.
- Trạng thái gồm `Active` và `Inactive`.

### Thêm người dùng

1. Bấm **Thêm mới**.
2. Nhập username, mật khẩu, họ tên, email, mã nhân viên, phòng ban, chức danh, trạng thái.
3. Lưu.
4. Điều kiện:
   - Username chỉ gồm chữ, số, `.`, `_`, `-`.
   - Mật khẩu tối thiểu 8 ký tự, có chữ thường, chữ hoa, số và ký tự đặc biệt.
   - Username, email, mã nhân viên không được trùng.

### Sửa hồ sơ

1. Chọn đúng một người dùng → **Chỉnh sửa**.
2. Username bị khóa; sửa thông tin hồ sơ, phòng ban, chức danh và trạng thái.
3. Chọn `Inactive` để ngừng hoạt động; chọn `Active` để áp dụng lại.
4. Lưu; người bị ngừng hoạt động không còn là actor hợp lệ trong các kiểm tra workflow.

### Chỉnh sửa tài khoản

1. Chọn đúng một người → **Chỉnh sửa tài khoản** hoặc biểu tượng bánh răng người dùng.
2. Cập nhật thông tin, nhóm tài khoản, ghi chú và trạng thái.
3. Có thể nhập mật khẩu mới; để trống thì giữ nguyên.
4. Khi lưu, hệ thống cập nhật hồ sơ → tài khoản/nhóm → mật khẩu nếu có.

### Reset mật khẩu

1. Chọn đúng một người → **Reset mật khẩu**.
2. Nhập và xác nhận mật khẩu mới theo chính sách.
3. Backend admin reset mật khẩu của tài khoản.

### Xóa

1. Chọn một hoặc nhiều người dùng → **Xóa** → xác nhận.
2. UI gọi xóa lần lượt và báo số bản ghi thành công/thất bại.
3. Đây là xóa tài khoản, khác với `Inactive`; khi chỉ muốn khóa đăng nhập nên dùng trạng thái `Inactive`.

## 7. Danh mục tri thức

**Mục đích:** quản lý cây phân loại tri thức và chuyên gia phụ trách.

**Quyền:** `CAT:VIEW`, `CAT:CREATE`, `CAT:UPDATE`, `CAT:DELETE`.

### Xem/tìm kiếm

- Mặc định chỉ hiển thị danh mục đang áp dụng.
- Bật **Hiển thị danh mục ngừng áp dụng** để xem cả hai trạng thái.
- Có thể mở từng nút để tải danh mục con; khi tìm kiếm, hệ thống tìm toàn cây.

### Thêm

1. Bấm **Thêm mới**.
2. Hệ thống sinh mã dạng `DM00<n>`; mã không cho nhập tay.
3. Nhập tên, mô tả, danh mục cha nếu có, chuyên gia phụ trách bắt buộc.
4. Lưu; mã danh mục phải duy nhất.

### Sửa

1. Chọn đúng một dòng hoặc nhấp đúp.
2. Sửa tên, mô tả, chuyên gia phụ trách/trạng thái hiển thị trên form.
3. Backend hiện cập nhật tên, mô tả, chuyên gia; không cập nhật mã. Việc đổi cha từ DTO hiện chưa được xử lý trong action Update.

### Ngừng áp dụng

1. Chọn một hoặc nhiều danh mục → **Ngừng áp dụng**.
2. Hệ thống ngừng áp dụng danh mục được chọn và toàn bộ danh mục con.
3. Bản ghi và tài liệu liên kết vẫn được giữ; danh mục không còn xuất hiện trong các dropdown chỉ lấy danh mục active.

### Áp dụng lại

1. Bật hiển thị danh mục ngừng áp dụng.
2. Chọn danh mục → **Kích hoạt**.
3. Hệ thống kích hoạt các mục được chọn; code không tự kích hoạt toàn bộ con hoặc cha, vì vậy cần chọn đủ các nút mong muốn.

### Xóa

1. Chọn một hoặc nhiều danh mục → **Xóa** → xác nhận.
2. Không xóa được nếu bất kỳ danh mục nào đang có tài liệu liên kết hoặc có danh mục con.
3. Muốn xóa cây: xử lý tài liệu liên kết và xóa từ lá lên gốc.

## 8. Không gian tri thức

**Mục đích:** tạo khu vực cộng tác theo phòng ban, dự án, cá nhân hoặc cộng đồng.

**Quyền:** `WS:VIEW`, `WS:CREATE`, `WS:UPDATE`, `WS:DELETE`; quyền thao tác thực tế còn phụ thuộc chủ sở hữu/thành viên.

### Xem và phạm vi truy cập

- Không gian công khai: người có quyền xem có thể truy cập.
- Không gian riêng tư: chỉ chủ sở hữu hoặc thành viên.
- Chỉ chủ sở hữu được sửa/xóa không gian.
- Chủ sở hữu hoặc thành viên có thể thực hiện một số thao tác đóng góp/thành viên theo backend hiện tại.

### Thêm

1. Bấm **Thêm không gian**.
2. Nhập tên, loại, mô tả, riêng tư/công khai, thành viên ban đầu.
3. Lưu; người tạo là Owner và được tính là thành viên mặc định.
4. Thành viên được gán role mặc định của workspace.

### Sửa

1. Chủ sở hữu chọn **Sửa**.
2. Cập nhật tên, loại, mô tả, quyền riêng tư và danh sách thành viên.
3. Khi lưu, thành viên mới được thêm; thành viên bị bỏ chọn được gỡ. Không thể gỡ Owner.

### Xóa

1. Chủ sở hữu chọn **Xóa** → xác nhận.
2. Backend xóa workspace; cần test thêm hành vi khi workspace đã có tài liệu do controller chưa có kiểm tra nghiệp vụ rõ ràng trước khi xóa.

### Thành viên

1. Mở chi tiết không gian → tab thành viên.
2. Thêm một/nhiều người chưa có trong không gian.
3. Chọn một/nhiều thành viên để xóa; Owner không được xóa.

### Tài liệu và thảo luận trong không gian

1. Mở tab tài liệu → **Thêm tài liệu mới**.
2. Chọn file, tiêu đề, tóm tắt, loại, danh mục, nguồn, tag, ngày hiệu lực/hết hạn.
3. Tài liệu có `workspaceId`: backend đặt trạng thái `Space`, phiên bản được duyệt ngay, không chạy workflow phê duyệt thường.
4. Có thể mở tài liệu, tải file và mở thảo luận.
5. Trong thảo luận, gửi bình luận hoặc trả lời; danh sách bỏ qua comment đã xóa.

### Import/Export

- Tải file mẫu Excel, điền dữ liệu rồi import hàng loạt.
- Dòng thiếu tên không gian bị bỏ qua; hệ thống báo các dòng lỗi/thành công.
- Export lấy toàn bộ danh sách theo bộ lọc hiện tại và sinh Excel.

## 9. Kho tài liệu

**Mục đích:** kho tra cứu trung tâm cho tài liệu người dùng được phép xem.

**Quyền:** xem `DOC:VIEW`; tải lên `DOC:UPLOAD`; sửa `DOC:EDIT`; xóa `DOC:DELETE`; quyền còn được resolve theo phạm vi tài liệu.

### Tìm và xem

1. Tìm kiếm theo từ khóa; lọc danh mục, đánh giá, ngày tạo và trạng thái hiển thị.
2. Mở chi tiết để xem metadata, preview, lịch sử phiên bản, thông tin duyệt và góp ý đã xử lý.
3. Các thao tác khả dụng: tải, đánh giá, góp ý, chia sẻ, sửa tùy quyền/phạm vi.

### Thêm tài liệu

1. Bấm **Thêm tài liệu mới**.
2. Chọn file; nhập tiêu đề, tóm tắt, loại tài liệu, danh mục, tag, nguồn cụ thể, ngày hiệu lực/hết hạn.
3. Hệ thống tính SHA-256 và có API kiểm tra trùng tên/hash.
4. Luồng lưu gồm: tạo Document → khởi tạo upload → PUT file vào object storage → finalize version.
5. UI hiện gửi `submitForApproval = true`; finalize sẽ tạo workflow nếu resolve được cấu hình.
6. Kết quả mong đợi: Document/Version ở `Pending`; xuất hiện trong hàng chờ của đúng người duyệt.

### Sửa/Update

1. Mở chi tiết → **Sửa**.
2. UI chi tiết hiện cập nhật metadata: tiêu đề, tóm tắt, tag, nguồn và ngày hiệu lực/hết hạn; loại và danh mục bị khóa.
3. Backend có API tạo/upload phiên bản mới. Về nghiệp vụ, một file thay thế phải tạo `DocumentVersion` mới và gửi workflow riêng; bản đã duyệt cũ vẫn là current version cho tới khi bản mới được duyệt.
4. UI hiện chưa cung cấp nhất quán luồng “thay file → tạo version → gửi duyệt lại”; không nên xem thao tác sửa metadata hiện tại là thay thế file.

### Tải, đánh giá, góp ý, chia sẻ

- **Tải:** lấy URL presigned của current version rồi tải trực tiếp từ storage; tăng download count.
- **Đánh giá:** mỗi người upsert điểm 1–5 và ghi chú; cập nhật điểm trung bình/số lượt.
- **Góp ý:** bắt buộc current version và nội dung; tạo Feedback trạng thái `New`.
- **Chia sẻ:** chia cho user/phòng ban theo mức quyền và có thể có thời hạn; tăng share count.

### Xóa

- Backend có `DELETE Documents/Delete/{id}` và kiểm tra `DOC:DELETE` theo phạm vi.
- UI Kho tài liệu hiện không trình bày nút xóa trong danh sách chính.
- Hàm client `deleteDocument(id)` hiện gọi URL thiếu `/{id}`; cần sửa trước khi đưa chức năng xóa lên UI.

## 10. Tài liệu của tôi

**Mục đích:** gom các tài liệu liên quan trực tiếp tới người dùng hiện tại.

**Quyền route:** `MD:VIEW`; catalog còn có `MD:UPLOAD`, `MD:UPDATE`, `MD:DELETE`.

### Ba tab nghiệp vụ

1. **Tài liệu đã upload**: tài liệu của người dùng, loại trừ `Archived`, `Draft`, `WaitingToSendCollect` theo query backend hiện tại.
2. **Được chia sẻ với tôi**: tài liệu được chia sẻ còn hiệu lực; hiển thị người chia sẻ.
3. **Tài liệu đã góp ý**: các Feedback do chính người dùng tạo.

### Thao tác

- Tab upload: xem và tải file.
- Tab được chia sẻ: xem, tải, đánh giá và chia sẻ tiếp nếu phạm vi cho phép.
- Tab góp ý: xem chi tiết góp ý, mở tài liệu, xóa góp ý.
- Chỉ xóa được góp ý của chính mình khi trạng thái còn `New`; đã xử lý/từ chối thì không xóa được.

### Lưu ý theo code hiện tại

- Component có code sửa tài liệu nhưng template không hiển thị nút sửa.
- Hàm xóa tài liệu mới chỉ `console.log`, chưa gọi API.
- Vì vậy luồng thực tế của màn này hiện là xem/tải/chia sẻ/đánh giá và quản lý góp ý; tạo/sửa/xóa tài liệu nên thực hiện ở Kho tài liệu cho tới khi UI được hoàn thiện.

## 11. Thu thập tri thức

**Mục đích:** tiếp nhận file từ nguồn ngoài rồi chuẩn hóa thành tài liệu KMS.

**Quyền route:** `CK:VIEW`; catalog có `CK:CREATE`, `CK:UPDATE`, `CK:DELETE`, `CK:IMPLEMENT`. Controller hiện đặt `CK:VIEW` ở cấp lớp nên cần rà lại enforcement chi tiết trước nghiệm thu.

### Trạng thái

```text
NotClassified → Classified → SentApproval → Approved
```

| Trạng thái | Thao tác chính |
| --- | --- |
| Chưa phân loại | Phân loại |
| Đã phân loại | Gửi duyệt |
| Đã gửi duyệt | Chờ màn Phê duyệt xử lý; nút bị khóa |
| Đã duyệt | Hoàn tất; nút bị khóa |

### Tải lên

1. Bấm **Tải lên tài liệu từ nguồn ngoài**.
2. Chọn/drag-drop một hoặc nhiều file.
3. Chọn nguồn (Email, SharePoint, OneDrive, Google Drive, Local, Other), nhập chi tiết nguồn; tác giả và nhóm nếu có.
4. Hệ thống tính hash và cảnh báo file trùng.
5. Tạo hàng loạt bản ghi Collect → khởi tạo upload từng file → upload song song → finalize.
6. Kết quả: file ở `NotClassified`.

### Phân loại

1. Ở file chưa phân loại, bấm **Phân loại**.
2. Nhập tiêu đề, tóm tắt, loại tài liệu, danh mục, tag, ngày hiệu lực/hết hạn và ghi chú.
3. Hệ thống yêu cầu file đã finalize hợp lệ.
4. Kết quả: tạo Document + Version liên kết bằng `ExternalId`; Document ở `WaitingToSendCollect`, Collect ở `Classified`.

### Gửi duyệt

1. Ở file đã phân loại, bấm **Gửi duyệt**.
2. Hệ thống tìm Document/Version liên kết và resolve workflow.
3. Thành công: workflow `Pending`, Document `Pending`, Collect `SentApproval`.
4. Thất bại nếu chưa phân loại, thiếu version hoặc không có workflow/rule phù hợp.

### Lưu ý theo code hiện tại

- Chưa có UI/API sửa hoặc xóa Collect sau khi tải lên.
- Enum có `Approved`, nhưng chưa thấy logic đồng bộ Collect từ `SentApproval` sang `Approved` khi workflow hoàn tất. Đây là điểm cần dev sửa hoặc BA xác nhận trạng thái nguồn có cần đồng bộ hay không.

## 12. Thu thập tri thức từ chuyên gia nghỉ hưu

**Mục đích:** ghi nhận tri thức bàn giao của chuyên gia sắp nghỉ hưu cùng tài liệu đính kèm.

**Quyền:** `RE:VIEW`, `RE:CREATE`, `RE:UPDATE`, `RE:DELETE`.

### Thêm

1. Bấm **Thêm mới**.
2. Chọn chuyên gia; phòng ban tự lấy từ hồ sơ.
3. Danh mục gốc tri thức chuyên gia do hệ thống xác định; chọn/nhập loại, nguồn dữ liệu, tiêu đề, tóm tắt, tag, ngày hiệu lực/hết hạn và file.
4. Luồng lưu: tạo hồ sơ + Document → upload/finalize Version → gửi workflow nếu `SubmitForApproval`.
5. Kết quả hiển thị trạng thái từ Document: chờ duyệt/đã duyệt/từ chối/hết hạn/lưu trữ.

### Sửa

1. Chọn bản ghi → **Sửa**.
2. Cập nhật chuyên gia, loại, nguồn và metadata Document.
3. Nếu Document đã `Approved`, backend đưa về `Draft` và cố gắng submit lại current version; nếu không tìm được version thì chuyển `Pending` bằng fallback.
4. Lưu thành công nhưng cần kiểm tra hàng chờ phê duyệt khi sửa bản đã duyệt.

### Xóa

1. Chọn bản ghi → **Xóa** → xác nhận.
2. Không thể xóa bản ghi có Document đã `Approved`.
3. Với bản chưa duyệt, hệ thống xóa hồ sơ và Document liên quan, giảm số tài liệu của workspace nếu có.

## 13. Cộng tác tri thức (Công tác tri thức)

**Mục đích:** tiếp nhận và xử lý góp ý trên phiên bản tài liệu.

**Quyền:** xem `CB:VIEW`; tạo góp ý qua quyền xem tài liệu; xử lý/từ chối `CB:HANDLE`.

### Tạo góp ý

1. Từ chi tiết tài liệu, chọn **Góp ý**.
2. Nhập nội dung bắt buộc và gửi.
3. Feedback gắn với current `DocumentVersion`, người tạo hiện tại và trạng thái `New`.

### Danh sách xử lý

1. Vào **Cộng tác tri thức**.
2. Tìm theo tài liệu/nội dung, người góp ý; lọc trạng thái và danh mục.
3. Mở chi tiết góp ý hoặc tài liệu gốc.

### Xác nhận đã xử lý

1. Với `New`/`InProgress`, chọn **Phê duyệt / xử lý góp ý**.
2. Nhập ghi chú xử lý nếu cần → xác nhận.
3. Feedback thành `Resolved`, ghi `ResolvedAt`; hệ thống phát sự kiện gửi email cho người góp ý.

### Từ chối

1. Với `New`/`InProgress`, chọn **Từ chối**.
2. Nhập lý do bắt buộc → xác nhận.
3. Feedback thành `Rejected`, lưu lý do/thời điểm và gửi email.

### Xóa góp ý

- Người tạo có thể xóa góp ý của chính mình ở tab **Tài liệu đã góp ý**.
- Chỉ xóa được khi trạng thái `New`; backend từ chối nếu đã xử lý/từ chối.
- `InProgress` tồn tại trong domain nhưng code hiện tại chưa có API/UI chuyển từ `New` sang `InProgress`.

## 14. Tài liệu hết hạn

**Mục đích:** theo dõi tài liệu hết hạn hoặc sắp hết hạn trong 30 ngày, thực hiện gia hạn hoặc lưu trữ.

**Quyền menu:** `ED:VIEW`. Backend truy vấn dùng `DOC:VIEW`, gia hạn dùng `DOC:EDIT`; catalog nghiệp vụ có `ED:EXTEND`, `ED:STORAGE` nhưng controller hiện chưa dùng các mã này nhất quán.

### Danh sách

- Màn query tài liệu có `ExpiryDate <= hôm nay + 30 ngày`.
- Có thể tìm kiếm, lọc danh mục, đánh giá, trạng thái và không gian.
- Xem chi tiết và tải phiên bản tài liệu.

### Gia hạn

1. Chọn tài liệu không phải `Archived` → **Gia hạn**.
2. Nhập ngày hết hạn mới, bắt buộc lớn hơn hôm nay.
3. Nếu tài liệu đang `Expired`, hệ thống đặt lại `Approved`.
4. Tăng bộ đếm số tài liệu đã gia hạn của người thao tác.

### Lưu trữ

1. Chọn tài liệu đã hết hạn → **Lưu trữ** → xác nhận.
2. Chỉ thực hiện nếu status `Expired` hoặc ngày hết hạn không lớn hơn hôm nay.
3. Kết quả: `Archived`, tăng bộ đếm lưu trữ; UI không cho gia hạn nữa.
4. Xác nhận UI nói không thể hoàn tác và code màn hiện không cung cấp thao tác khôi phục.

### Tự động hết hạn

- Worker `DocumentExpiryJob` chịu trách nhiệm đánh dấu tài liệu hết hạn theo lịch chạy.
- Cần test cả hai trường hợp: status đã là `Expired` và ngày hết hạn đã qua nhưng job chưa chạy; query vẫn phải đưa vào danh sách phù hợp.

## 15. AI Agent

**Mục đích:** hỏi đáp trên kho tri thức, xem lịch sử, thống kê và khoảng trống tri thức.

**Quyền tách theo chức năng:** `AI:CHAT`, `AI:HISTORY`, `AI:DASHBOARD`, `AI:GAPS`; widget dùng `AI:WIDGET`.

### 15.1. Trò chuyện AI

1. Chọn ngôn ngữ phản hồi.
2. Nhập câu hỏi và gửi; không gửi chuỗi rỗng hoặc vượt giới hạn ký tự.
3. Nếu chưa có thread, hệ thống tạo thread; câu trả lời được stream về giao diện.
4. Tài liệu tham chiếu trong câu trả lời có thể được mở/tải theo `documentId`.
5. Người dùng có thể sao chép, Like/Dislike; khi Dislike có thể thêm hoặc sửa feedback/nhãn.
6. **Cuộc hội thoại mới** tạo ngữ cảnh mới.

### 15.2. Lịch sử hội thoại

1. Tìm theo tiêu đề/người dùng; lọc khoảng ngày.
2. Mở dòng để xem các message.
3. Có thể đổi tên thread.
4. Xóa thread sau xác nhận; danh sách thống kê cả số thread đã xóa khi API cho phép.

### 15.3. Dashboard AI

- Lọc theo khoảng ngày → **Áp dụng** hoặc **Đặt lại**.
- Xem tổng thread/message/user, message theo ngày, Like/Dislike và phân bố lý do feedback.
- Đây là màn chỉ đọc; có trạng thái không quyền, đang tải, lỗi/thử lại và không có dữ liệu.

### 15.4. Khoảng trống tri thức

1. Tìm theo trace/câu hỏi/câu trả lời; lọc nhãn và ngày.
2. Mở chi tiết trace để phân tích câu hỏi–câu trả lời.
3. Chọn một trace hoặc nhiều trace → xóa → xác nhận.
4. Xóa gọi lần lượt API; cần kiểm tra báo cáo một phần thành công nếu có lỗi.

## 16. Phê duyệt tài liệu

**Mục đích:** xử lý workflow của tài liệu đào tạo và tài liệu công ty.

**Quyền:** vào hàng chờ `WF:VIEW_QUEUE`; duyệt/yêu cầu sửa `WF:APPROVE`; từ chối `WF:REJECT`. Ngoài RBAC, actor phải khớp rule của bước hiện tại.

### Xem hàng chờ

1. Chọn loại tài liệu đào tạo/công ty hoặc tất cả.
2. Lọc bước duyệt, trạng thái, danh mục; tìm theo từ khóa.
3. Hệ thống chỉ trả những phiếu người hiện tại có thể xem/xử lý, cộng với phiếu đã hoàn thành mà họ là người gửi hoặc từng thao tác.
4. Mở thông tin tài liệu ở chế độ chỉ xem; mở chi tiết phê duyệt để xem step và lịch sử log.

### Phê duyệt

1. Mở phiếu `Pending`; nhập nhận xét nếu cần.
2. Bấm **Phê duyệt**.
3. Nếu còn bước kế tiếp: bước hiện tại `Approved`, bước sau `Active`, workflow vẫn `Pending`.
4. Nếu là bước cuối: Workflow, Version và Document thành `Approved`; version được đặt làm current version, ghi người/thời gian duyệt.

### Từ chối

1. Mở phiếu `Pending` → **Từ chối**.
2. Gửi nhận xét/lý do.
3. Bước hiện tại, workflow, version và document chuyển `Rejected`; workflow kết thúc.

### Yêu cầu chỉnh sửa

1. Mở phiếu `Pending` → **Yêu cầu sửa**.
2. Kết quả: workflow, version, document về `Draft`; bước runtime dùng trạng thái `Rejected`; workflow kết thúc.
3. Chủ tài liệu phải sửa/tạo version phù hợp rồi gửi duyệt lại.

### Điều kiện bị chặn

- Phiếu không còn `Pending` hoặc đã có `EndedAt`.
- Người dùng inactive/không tồn tại.
- Không khớp chuyên gia danh mục, chức danh hoặc chiến lược phòng ban của rule.
- Thiếu quyền workflow tương ứng.
- Step không phải chế độ `AnyOne`.
- Workflow/rule không active hoặc không còn hợp lệ.

### Audit

- Mỗi Submit/Approve/Reject/RequestChange thành công được ghi log actor, phòng ban, chức danh, thời gian và comment.
- Các lần action bị từ chối sau khi đã xác định instance/actor cũng được ghi log thất bại cùng reason code.

## 17. Ma trận thao tác nhanh

| Màn hình | Thêm | Sửa/Update | Xóa | Ngừng/Áp dụng | Duyệt/Xử lý | Khác |
| --- | --- | --- | --- | --- | --- | --- |
| Báo cáo & phân tích | — | — | — | Áp dụng bộ lọc | — | Export Excel, drill-down |
| Phân quyền | Nhóm | Nhóm/quyền/thành viên | Nhóm có điều kiện | Active/Inactive nhóm | — | Gán quyền theo danh mục–phòng ban |
| Quản lý người dùng | Có | Hồ sơ/tài khoản | Có | Active/Inactive | — | Reset mật khẩu |
| Danh mục tri thức | Có | Có | Có điều kiện | Có | — | Cây cha–con |
| Không gian tri thức | Có/import | Có | Owner | Chưa có UI lifecycle | — | Thành viên, tài liệu, thảo luận, export |
| Kho tài liệu | Có | Có/phiên bản mới | Backend có, UI chưa hoàn chỉnh | — | Gửi workflow | Tải, share, rating, feedback |
| Tài liệu của tôi | — | Code có nhưng UI chưa mở | Xóa tài liệu chưa làm; xóa feedback có | — | — | 3 tab cá nhân |
| Thu thập tri thức | Upload | Phân loại | Chưa có | — | Gửi duyệt | Download nguồn |
| Chuyên gia nghỉ hưu | Có | Có, duyệt lại khi cần | Không xóa bản đã duyệt | — | Gửi workflow | File đính kèm |
| Cộng tác tri thức | Tạo góp ý | Ghi chú xử lý | Chủ góp ý, chỉ trạng thái New | — | Resolve/Reject | Gửi email |
| Tài liệu hết hạn | — | Gia hạn | — | — | — | Lưu trữ |
| AI Agent | Thread/chat | Đổi tên, feedback | Thread/trace | Áp dụng bộ lọc | — | Dashboard, khoảng trống |
| Phê duyệt tài liệu | Submit từ màn nguồn | Yêu cầu sửa | — | — | Approve/Reject | Audit log |

## 18. Checklist test nghiệp vụ tối thiểu

### CRUD và phân quyền

- [ ] Tài khoản không có quyền không thấy menu và gọi API bị 403.
- [ ] Thêm mới thành công với dữ liệu hợp lệ; trùng mã/username/email/hash trả thông báo phù hợp.
- [ ] Sửa chỉ thay đổi trường cho phép; trường khóa không bị thay đổi bằng cách gọi API trực tiếp.
- [ ] Xóa có xác nhận; các liên kết đang dùng chặn xóa đúng quy tắc.
- [ ] Active/Inactive thay đổi khả năng đăng nhập/chọn trong dropdown/phê duyệt đúng mong đợi.
- [ ] Thao tác hàng loạt báo đúng thành công toàn phần, một phần và thất bại.

### Tài liệu và file

- [ ] Tạo Document lỗi ở bất kỳ bước upload/finalize nào không để lại dữ liệu/file mồ côi hoặc có cơ chế xử lý lại.
- [ ] File rỗng, MIME không hỗ trợ, link presigned hết hạn và file không tồn tại cho thông báo đúng.
- [ ] Phiên bản mới không thay thế current version trước khi được duyệt.
- [ ] Tài liệu không gian không đi vào hàng chờ workflow thường.
- [ ] Quyền danh mục–phòng ban được áp dụng cho xem, tải, chia sẻ và sửa.

### Workflow

- [ ] Gửi duyệt khi thiếu workflow/rule trả lỗi rõ ràng và không đổi sai trạng thái.
- [ ] Người không đúng chuyên gia/chức danh/phòng ban không thấy hoặc không action được phiếu.
- [ ] Duyệt bước giữa mở đúng bước tiếp theo; duyệt cuối cập nhật Document và Version.
- [ ] Reject và RequestChange kết thúc workflow với trạng thái khác nhau.
- [ ] Không thể action hai lần hoặc action phiếu đã kết thúc.
- [ ] Lịch sử ghi đủ action thành công/thất bại và comment.

### Hết hạn và cộng tác

- [ ] Ngày gia hạn phải lớn hơn hôm nay; Archived không gia hạn được.
- [ ] Chỉ tài liệu thực sự hết hạn được lưu trữ.
- [ ] Chỉ chủ góp ý xóa được feedback `New`.
- [ ] Resolve/Reject gửi email và lưu ghi chú/thời điểm.

## 19. Các điểm cần BA/Dev xác nhận trước nghiệm thu

1. **Thu thập tri thức:** chưa đồng bộ `SentApproval → Approved` sau khi workflow duyệt xong.
2. **Tài liệu của tôi:** nút sửa/xóa tài liệu chưa được nối vào UI; hàm xóa đang là stub.
3. **Kho tài liệu:** client xóa tài liệu đang gọi thiếu id và danh sách chưa có nút xóa.
4. **Cập nhật phiên bản tài liệu:** API backend có nhưng UI sửa ở Kho tài liệu chủ yếu chỉ cập nhật metadata; code sửa có upload ở Tài liệu của tôi lại chưa được mở trên template và không bật `submitForApproval`. Cần thống nhất luồng thay file/gửi duyệt lại.
5. **Danh mục:** UI cho chọn cha/trạng thái khi sửa nhưng backend Update không đổi cha và không dùng `isActive`; nên dùng action Activate/Deactivate riêng hoặc bổ sung backend.
6. **Không gian tri thức:** có enum `Active/Inactive` và quyền `WS:ARCHIVE`, nhưng chưa có luồng UI/API ngừng áp dụng/lưu trữ.
7. **Cộng tác:** chưa có action chuyển Feedback sang `InProgress` dù domain có trạng thái này.
8. **Tài liệu hết hạn:** quyền route dùng `ED:*` nhưng controller chủ yếu kiểm tra `DOC:*`; action lưu trữ chưa gắn attribute quyền chuyên biệt.
9. **Báo cáo:** route Angular chưa gắn `permissionGuard` trực tiếp, dù menu và backend dùng `DAS:VIEW`.
10. **Workflow:** domain có `All`/`Quorum` nhưng runtime chỉ hỗ trợ `AnyOne`.
11. **Xóa workspace có tài liệu:** backend chưa thể hiện quy tắc chặn/xử lý liên kết; cần thống nhất trước khi cho phép trên môi trường thật.

## 20. Tham chiếu API chính cho developer

| Nghiệp vụ | Controller/API chính |
| --- | --- |
| Người dùng | `Users/GetAll`, `Create`, `Update/{id}`, `UpdateAccount/{id}`, `Delete/{id}`, `Activate/{id}`, `Deactivate/{id}`, `AdminResetPassword/{id}` |
| Nhóm và quyền | `Groups/*`, `GroupMembers/*`, `Permissions/GetCatalog`, `RoleBindings/AssignMany`, `CategoryOrgPermissions/*` |
| Danh mục | `Categories/GetList`, `Create`, `Update/{id}`, `DeleteMany`, `Activate`, `Deactivate`, `GetChildren` |
| Không gian | `Workspace/*`, `WorkspaceMember/*` |
| Tài liệu | `Documents/Create`, `Update/{id}`, `Delete/{id}`, `InitializeUpload/{id}`, `FinalizeUpload/{id}`, `GetVersions/{id}`, `GetDownloadUrl/{id}` |
| Thu thập | `Collect/CreateDocumentCollects`, `InitializeUpload`, `FinalizeUpload`, `Classify`, `SendForApproval` |
| Chuyên gia nghỉ hưu | `CollectKnowledgeRetire/Create`, `Update/{id}`, `Delete/{id}` và upload/finalize |
| Góp ý | `Feedback/CreateFeedback`, `GetAllFeedback`, `ResolveFeedback`, `RejectFeedback`, `DeleteFeedback` |
| Hết hạn | `DocumentExpired/QueryList`, `GetExpiringSoonDocuments`, `Renew/{id}`, `ArchiveExpiredDocument` |
| Phê duyệt | `DocumentApprovals/QueryItems`, `QueryQueue`, `QueryFacets`, `GetVisibleSteps`, `Approve/{id}`, `Reject/{id}`, `RequestChange/{id}`, `GetLogs/{id}` |
| Báo cáo | `Reports/Overview`, `ActiveUserRanking`, `DepartmentRanking`, các API export Excel |
| AI Agent | Agent Manager `/api/v1/stats/summary`, `/api/v1/threads`, `/messages`, `/rating`, `/api/v1/analytics/traces` |

