# DLL
<https://github.com/TsudaKageyu/minhook>

```cpp
#include <windows.h>
#include <stdio.h>
#include "MinHook.h" // Cần include header của MinHook

// 1. Khai báo con trỏ hàm gốc (giống PDF)
typedef int (WINAPI* fnMessageBoxA)(HWND, LPCSTR, LPCSTR, UINT);
fnMessageBoxA g_pMessageBoxA = NULL;

// 2. Hàm Detour (Hàm thay thế) - Sẽ chạy thay cho MessageBoxA
INT WINAPI MyMessageBoxA(HWND hWnd, LPCSTR lpText, LPCSTR lpCaption, UINT uType) {
    // (Tùy chọn) Bạn có thể ghi log ở đây nếu muốn
    // printf("[+] Original lpText: %s\n", lpText); 
    
    // Gọi hàm gốc nhưng thay đổi nội dung (giống hệt PDF trang 6)
    return g_pMessageBoxA(hWnd, "HACKED: different lpText", "HACKED: different lpCaption", uType);
}

// 3. Hàm cài đặt Hook
BOOL InstallHook() {
    // Khởi tạo MinHook
    if (MH_Initialize() != MH_OK) return FALSE;

    // Tạo Hook cho hàm MessageBoxA
    if (MH_CreateHook(&MessageBoxA, &MyMessageBoxA, 
                      reinterpret_cast<LPVOID*>(&g_pMessageBoxA)) != MH_OK) {
        return FALSE;
    }

    // Kích hoạt Hook
    if (MH_EnableHook(&MessageBoxA) != MH_OK) {
        return FALSE;
    }
    return TRUE;
}

// 4. Hàm gỡ bỏ Hook (Khi DLL bị đẩy ra khỏi tiến trình)
void RemoveHook() {
    MH_DisableHook(&MessageBoxA);
    MH_Uninitialize();
}

// 5. Điểm vào của DLL (DllMain)
BOOL APIENTRY DllMain(HMODULE hModule, DWORD ul_reason_for_call, LPVOID lpReserved) {
    switch (ul_reason_for_call) {
        case DLL_PROCESS_ATTACH:
            // Khi DLL được Inject vào tiến trình, ta tiến hành Hook ngay
            InstallHook();
            break;
        case DLL_PROCESS_DETACH:
            // Khi DLL bị gỡ ra, ta dọn dẹp
            RemoveHook();
            break;
    }
    return TRUE;
}
```

Cấu hình Visual Studio để build thành công:

Include thư mục MinHook:

Chuột phải vào Project -> Properties.

Vào C/C++ -> General -> Additional Include Directories.

Thêm đường dẫn tới thư mục chứa file MinHook.h (thường là include trong thư mục MinHook bạn tải về).

Link tới file .lib:

Vẫn trong Properties, vào Linker -> General -> Additional Library Directories.

Thêm đường dẫn tới thư mục chứa file libMinHook.x64.lib (đã build ở Bước 1).

Vào tiếp Linker -> Input -> Additional Dependencies, thêm dòng chữ: libMinHook.x64.lib (hoặc x86 nếu bạn chọn x86).

Build: Chọn Release và x64, sau đó ấn Build (Ctrl+Shift+B). Bạn sẽ có file HookMessageBox.dll trong thư mục Release.

```cpp
#include <windows.h>
#include <tlhelp32.h>
#include <iostream>
#include <string>

// Hàm hỗ trợ: Lấy PID từ tên Process (Process Identification)
DWORD GetProcessIdByName(const char* processName) {
    PROCESSENTRY32 processEntry;
    processEntry.dwSize = sizeof(PROCESSENTRY32);

    HANDLE snapshot = CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, 0);
    if (snapshot == INVALID_HANDLE_VALUE) return 0;

    if (Process32First(snapshot, &processEntry)) {
        do {
            if (_stricmp(processEntry.szExeFile, processName) == 0) {
                DWORD pid = processEntry.th32ProcessID;
                CloseHandle(snapshot);
                return pid;
            }
        } while (Process32Next(snapshot, &processEntry));
    }
    CloseHandle(snapshot);
    return 0; // Không tìm thấy
}

// Hàm chính: Thực hiện DLL Injection
bool InjectDLL(DWORD pid, const char* dllPath) {
    // 1. Mở tiến trình đích (Open Target Process)
    HANDLE hProcess = OpenProcess(PROCESS_ALL_ACCESS, FALSE, pid);
    if (hProcess == NULL) {
        std::cout << "[-] Khong the mo Process. Loi: " << GetLastError() << std::endl;
        return false;
    }

    // 2. Phân giải đường dẫn DLL thành đường dẫn tuyệt đối (DLL Path Resolution)
    char fullDllPath[MAX_PATH];
    if (GetFullPathNameA(dllPath, MAX_PATH, fullDllPath, NULL) == 0) {
        std::cout << "[-] Duong dan DLL khong hop le." << std::endl;
        CloseHandle(hProcess);
        return false;
    }
    
    // Cấp phát bộ nhớ trong tiến trình đích cho đường dẫn DLL
    size_t dllPathSize = strlen(fullDllPath) + 1;
    LPVOID pRemoteMemory = VirtualAllocEx(hProcess, NULL, dllPathSize, MEM_COMMIT | MEM_RESERVE, PAGE_READWRITE);
    if (pRemoteMemory == NULL) {
        std::cout << "[-] Cap phat bo nho that bai." << std::endl;
        CloseHandle(hProcess);
        return false;
    }

    // Ghi đường dẫn DLL vào bộ nhớ của tiến trình đích
    if (!WriteProcessMemory(hProcess, pRemoteMemory, fullDllPath, dllPathSize, NULL)) {
        std::cout << "[-] Ghi bo nho that bai." << std::endl;
        VirtualFreeEx(hProcess, pRemoteMemory, 0, MEM_RELEASE);
        CloseHandle(hProcess);
        return false;
    }

    // 3. Tạo Remote Thread và gọi LoadLibraryA (Remote Thread Creation)
    // Lấy địa chỉ hàm LoadLibraryA từ kernel32.dll
    HMODULE hKernel32 = GetModuleHandleA("kernel32.dll");
    LPTHREAD_START_ROUTINE loadLibraryAddr = (LPTHREAD_START_ROUTINE)GetProcAddress(hKernel32, "LoadLibraryA");
    
    if (loadLibraryAddr == NULL) {
        std::cout << "[-] Khong tim thay LoadLibraryA." << std::endl;
        VirtualFreeEx(hProcess, pRemoteMemory, 0, MEM_RELEASE);
        CloseHandle(hProcess);
        return false;
    }

    // Tạo Thread trong tiến trình đích
    HANDLE hThread = CreateRemoteThread(hProcess, NULL, 0, loadLibraryAddr, pRemoteMemory, 0, NULL);
    if (hThread == NULL) {
        std::cout << "[-] Tao Remote Thread that bai. Loi: " << GetLastError() << std::endl;
        VirtualFreeEx(hProcess, pRemoteMemory, 0, MEM_RELEASE);
        CloseHandle(hProcess);
        return false;
    }

    // Chờ thread chạy xong (DLL load xong)
    WaitForSingleObject(hThread, INFINITE);

    // Dọn dẹp
    CloseHandle(hThread);
    VirtualFreeEx(hProcess, pRemoteMemory, 0, MEM_RELEASE);
    CloseHandle(hProcess);

    return true;
}

int main(int argc, char* argv[]) {
    if (argc != 3) {
        std::cout << "Cach su dung:\n";
        std::cout << "1. Theo Ten Process: Injector.exe <ten_process.exe> <duong_dan_dll>\n";
        std::cout << "2. Theo PID:         Injector.exe <PID> <duong_dan_dll>\n";
        return 1;
    }

    std::string target = argv[1];
    const char* dllPath = argv[2];
    DWORD pid = 0;

    // [1] Process Identification: Kiểm tra xem input là PID (số) hay Tên (chữ)
    bool isNumeric = true;
    for (char c : target) {
        if (!isdigit(c)) { isNumeric = false; break; }
    }

    if (isNumeric) {
        pid = std::stoul(target);
        std::cout << "[+] Target PID: " << pid << std::endl;
    } else {
        pid = GetProcessIdByName(target.c_str());
        if (pid == 0) {
            std::cout << "[-] Khong tim thay tien trinh: " << target << std::endl;
            return 1;
        }
        std::cout << "[+] Tim thay Process: " << target << " (PID: " << pid << ")" << std::endl;
    }

    // Tiến hành Inject
    std::cout << "[*] Dang Inject DLL: " << dllPath << " ..." << std::endl;
    if (InjectDLL(pid, dllPath)) {
        std::cout << "[+] Inject THANH CONG!" << std::endl;
    } else {
        std::cout << "[-] Inject THAT BAI!" << std::endl;
    }

    return 0;
}
```