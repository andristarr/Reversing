# Reversing Experiment #5

In this page I will be doing some reverse engineering on a small binary, that I have put together. Since this is made by me there is no legal problem with reversing.


## Overview

Malware tend to hide their tracks in whatever they are doing. They try to make static and dynamic analysis harder in order to stay under cover.
A reverser might look at the import table of the PE and see that the malware imports the LoadLibraryA Kernel32.dll function. That point would be a good idea to put a breakpoint to see, what else the malware imports in order to understand what it is doing.

This is why malware might use the following method to import Windows DLLs.

## PEB

PEB stands for Process Environment Block and it contains very useful information regarding the execution context. (You can read about PEB in detail here: https://docs.microsoft.com/en-us/windows/win32/api/winternl/ns-winternl-peb) PEB is not to be used by the end users and is mainly for internal use by the OS. It is officially not fully documented.

Now in the PEB structure we have the member
>PPEB_LDR_DATA Ldr;

This member contains a property to the PEB_LDR_DATA struct. Which contains information about the modules loaded by the PE. (Further reading here: https://docs.microsoft.com/en-us/windows/win32/api/winternl/ns-winternl-peb_ldr_data)

## Usage

If we walk this PEB_LDR_DATA we can find the modules loaded by the library again. On Windows whenever you run a PE certain DLLs get loaded automatically by Windows even if you don't specifically import them. This includes ntdll.dll, Kernel32.dll, Kernelbase.dll. Through the PEB structure we can get the address of these DLLs and use them to call WinAPI functions.
Malware use this method to import WinAPI functions dynamically without the usage of LoadLibrary in order to prevent easy debugging.

#### Note
After checking the Microsoft documentation they state that the DllBase is at _LDR_DATA_TABLE_ENTRY + 18h, but on my machine (build 19043) the DllBase is actually at _LDR_DATA_TABLE_ENTRY  + 10h, which can be addressed as:

>ldr->Reserved2[0]

# Signs
We can pretty easily spot if a malware is using this method to dynamically get addresses of WinAPI dlls. The TEB structure (Thread Environment Block. Further reading at: https://docs.microsoft.com/en-us/windows/win32/api/winternl/ns-winternl-teb) contains a pointer to the PEB. The TEB struct is always loaded into the fs segment register on Windows systems. Even though the 'filesystem' register was intended for other uses, Windows decided to utilise it this way.
Knowing the TEB is in the fs segment register a malware can use it to get a pointer address to the PEB. What you need look for in the code during reversing would look something like this:

> .text:0040167E                 mov     eax, large fs:30h

The C++ equivalent would be:

    DWORD ppeb;
    
    __asm {
    	mov eax, fs:30h	
    	mov ppeb, eax
    }

Then they can step through the Ldr to find the DLL they look for and get the address. In this example the ntdll is searched for and the address of the NtQuerySystemInformation function is collected.

    if (currentName== std::string("ntdll.dll")) // malware could use stack string to hide the hardcoded ntdll.dll value
    {
    		hNtdll = (HMODULE)(ldrEntry->Reserved2[0]);
    		pfnNtQuerySystemInformation fnNtQuerySystemInformation;
    		fnNtQuerySystemInformation = (pfnNtQuerySystemInformation)GetProcAddress(hNtdll, "NtQuerySystemInformation");
    }


# Conclusion
It is a good thing to keep in mind, if we don't see any imports from the malware to check whether they import the dynamic libraries in the above described way.