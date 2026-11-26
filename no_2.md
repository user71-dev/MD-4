# DLL
<https://github.com/TsudaKageyu/minhook>

Đây là chương trình Injector có nhiệm vụ inject DLL ta đã viết ở phần trước vào trong process

# Hàm main 
Đóng vai trò là entry point nhận lệnh từ user, cho phép user truyền vào theo hai option khác nhau: một là nhập bằng process name, hai là nhập bằng pid.

Giống như các phần khác, cấu trúc hàm main chỉ đơn giản như sau:
- Kiểm tra tham số đầu vào xem thử có valid hay không
- Nếu input là số thì cast thành số nguyên, còn nếu là chữ cái thì cast thành Unicode và gọi GetProcessIdByName() để tra ra pid
- Cuối cùng là gọi InjectDll() để inject và in kết quả

# Hàm GetProcessIdByName()
Logic tương tự như các module trước, hàm này quen thuộc cho nên ta sẽ chỉ nói sơ qua, không đi sâu vào logic.

Hàm take snapshot bằng thư viện tlhelp32, dùng so sánh không phân biệt chữ hoa thường.

# Hàm InjectDll()
Đây là phần core của Injector. Logic lần lượt của hàm từ trên xuống dưới như sau:
- Gọi Openprocess để mở target process
- Chuyển đường dẫn DLL sang đường dẫn tuyệt dối bằng hàm GetFullPathNameA
- Cấp phát bộ nhớ trong target process bằng VirtualAllocEx
- Ghi DLL paht vào vùng nhớ vừa cấp bằng WriteProcessMemory
- Lấy địa chỉ hàm LoadLibraryA bằng GetModuleHandleA và GetProcAddress
- Cuối cùng là tạo luồng từ xa bằng CreateRemoteThread, sau đó cleanup nếu hoàn thành xong.