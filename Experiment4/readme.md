# Reversing Experiment #4

In this page I will be doing some reverse engineering on a crackme, that is indicated as difficulty 2.7. These are made with the purpose of being reversed, so there is no legal constraints regarding that. The one, that I will be reversing can be found at: https://crackmes.one/crackme/62115f8133c5d46c8bcbfee3

### Note
In this crackme the aim is to find a flag in the executeable. I will show two solutions to finding the flag.

## Exploration

Let's see what the executeable does when ran:
>Untitled.exe<br />
>try harder ...<br />

It does nothing, but shows us the bad message.

Let's open it in PEBear. We can see that the executeable has the standard section names, so it doesn't seem to be packed. We can just open it in a disassembler.
It exports a 3 addresses, but none seems to be the C main entry point:
>TlsCallback_0	00000000004018C0	<br />
>TlsCallback_1	0000000000401890	<br />
>mainCRTStartup	00000000004014E0	[main entry]<br />

mainCRTStartup in this case probably stands for main C runtime startup, so it is bootstrapping the C runtime environment. But searching for the C main function does give us the C entry point to the exe.

Looking at that we can see that the main function loads a value from the data section of the PE file into eax and check to see, if that value is not zero. If it is zero, we get the bad message.

>.text:0000000000401550                 push    rbp	<br />
>.text:0000000000401551                 mov     rbp, rsp	<br />
>.text:0000000000401554                 sub     rsp, 20h	<br />
>.text:0000000000401558                 call    __main	<br />
>.text:000000000040155D                 mov     eax, cs:print_flag	<br />
>.text:0000000000401563                 test    eax, eax	<br />
>.text:0000000000401565                 jnz     short loc_401575	<br />
>.text:0000000000401567                 lea     rcx, Buffer     ; "try harder ..."	<br />
>.text:000000000040156E                 call    puts	<br />

If it's not zero, we do a short jump to address 0x401575. Let's see what's there. 
>.text:0000000000401586                 push    rbp<br />
>.text:0000000000401587                 mov     rbp, rsp<br />
>.text:000000000040158A                 sub     rsp, 30h<br />
>.text:000000000040158E                 mov     [rbp+arg_0], ecx<br />
>.text:0000000000401591                 lea     rax, aFlagDanof4edxF ; "flag{Danof4edx_flag}"<br />

We can see that the flag is hardcoded as a string into the file.

Based on the information above another way to find the flag would be as simple as doing a string search.  The flag string is picked up:
> .rdata:000000000040400F	00000015	C	flag{Danof4edx_flag}

Let's see if we are allowed to patch, we can also get the flag. Looking at instructions at .text:0000000000401565 we can see a jnz, that jumps based on the zero flag that is set in the test above. Essentially if the print_flag input variable is set to 0 it will show us the bad message.
Patching this instruction to a jz will always get us to the flag, to the good message. So modifying the opcodes from 0x75 0x0E to 0x74 0x0E will always direct us to the good message.
>Untitled.exe<br />
>flag{Danof4edx_flag}<br />



## Take aways

We could see, that this crackme was rather easy to solve, but it was a good candidate to explore different ways of finding the flag. (Analysing the code flow at the entry point, string search and patching the file).


## Credits

Crackmes.one and the community for providing fun challanges.