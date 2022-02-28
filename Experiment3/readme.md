
# Reversing Experiment #3

In this page I will be doing some reverse engineering on a crackme, that is indicated as difficulty 2.5. These are made with the purpose of being reversed, so there is no legal constraints regarding that. The one, that I will be reversing can be found at: https://crackmes.one/crackme/62122b9d33c5d46c8bcbff02



## Exploration

Let's see what the executeable does when ran:
>(1) - rules<br />
>(2) - start Training<br />
> Select an option : 2<br />
> Enter Your Username : qwert<br />
> Password : trewq<br />
>-> (1 / 10)<br />
>Wrong password for the username "qwert": ! Try again ...<br />
> Password :<br />

The crakcme loads and we can see it asks for a username and then prompts for a password, then based on those inputs returns whether the pair passes the tests. We get the bad message in this case. (Copied as is.)
>Wrong password for the username "qwert": ! Try again ...<br />

Let's open the executable in a disassembler. We will see that there is 3 sections. UPX0, UPX1, UPX2. As these sections are not part of the standard PE sections, it is straightforward to guess, that this executeable is packed, if we search for the referenced strings, we won't see the strings presented to us when we ran the program hardcoded. Let's open the executeable in x64dbg, maybe we can see more. After loading the exe we will see in the Memory Map our keygenme.exe PE. Searching for referenced strings in this module, we find the hardcoded strings. Meaning, now, that the crackme has been loaded to memory the packer unpacked itself and we can clearly see the strings.
>Address=0000000000401677<br />
>Disassembly=lea rcx,qword ptr ds:[405087]<br />
>String="Good Password ! "<br />

Clicking the string gets us to a function that either displays the good or the bad message. We could patch this function later, but first let's see what it does. Stepping through the debugger line by line we can see that the code basically steps through our entered string one-by-one. There is a call and a compare before the jump to the bad/good message. Let us open that call and put a breakpoint to the beginning of that function. We can see, that the function does some calls, we are interested in the x64 calling convetion's return value. It gets stored in RAX and we can see, that there is an address in RAX.
>0000000000401753   55   push rbp <br />
>Value of RAX at the end is: 00000000007D1700<br />

In the memory view, going to the value, that is stored in RAX we can find a string
> 00000000007D1700  65 74 65 74 71 AB AB AB AB AB AB AB AB AB AB AB  etetq«««««««««««  <br />

Maybe, this is the password that is calculated from the input and indeed it is correct.
> Enter Your Username : qwert<br />
> Password : etetq<br />
>Good Password !<br />

Since we want to make a keygen, it would be best to unpack the exe in order to preform static analisys on it. After some searching I have found a pretty good write-up on this.
> https://infosecwriteups.com/how-to-unpack-upx-packed-malware-with-a-single-breakpoint-4d3a23e21332<br />

Following this write-up you will be able to unpack the executeable. One thing to note, this is for x86 which has pushad and popad instructions. AMD64 removed those instructions, so we have:
>0000000000414DD0 53 push rbx<br />
>0000000000414DD1 56 push rsi<br />
>0000000000414DD2 57 push rdi<br />
>0000000000414DD3 55 push rbp<br />

And the reverse of this would be the pop sequence of these registers, we can easily find these in reverse order and also the jump to the OEP.
>0000000000414FE1 | 5D                       | pop rbp                                 |<br />
>0000000000414FE2 | 5F                       | pop rdi                                 |<br />
>0000000000414FE3 | 5E                       | pop rsi                                 |<br />
>0000000000414FE4 | 5B                       | pop rbx                                 |<br />
>0000000000414FE5 | 48:8D4424 80             | lea rax,qword ptr ss:[rsp-80]           | rax:EntryPoint<br />
>0000000000414FEA | 6A 00                    | push 0                                  |<br />
>0000000000414FEC | 48:39C4                  | cmp rsp,rax                             | rax:EntryPoint<br />
>0000000000414FEF | 75 F9                    | jne keygenme.414FEA                     |<br />
>0000000000414FF1 | 48:83EC 80               | sub rsp,FFFFFFFFFFFFFF80                |<br />
>0000000000414FF5 | E9 E6C4FEFF              | jmp keygenme.4014E0                     |<br />

So after unpacking the crackme we can open it up in a disassembler and go to the C main function. To speed things up, let's check the code in decompiler view. (Ghidra disassembler comes with a built in decompiler for free.)
The main function calls different functions based off of the value entered. If we enter 2, it will go to the enter serial prompt.
> int __fastcall ark_mainLoop(int a1)

I have done some reversing on the executeable and renamed some of the function names/variables. We can see, that it calls the printf and then scanf to read in the value entered for the username. It then calls a function which  I have named 'ark_GeneratePass' with an argument that contains the entered username. This is the function we checked in x64dbg whose return value in RAX stored the generated password. In order to create a keygen we will need to understand this functions logic.
> GeneratedPass = ark_GeneratePass(ark_ReadlUserName);

Based in the return value of this function, we call a comparison function, that checks the length and each byte of the strings to match.

In the ark_GeneratePass function we see that we get the length of a hardcoded string
> tryharderLength = get_Length((__int64)aTryhardertomak);<br />
 > usernameLength = get_Length(username);<br />

We create a new string in memory and then start filling it based on a specific logic that can be see after renaming the variables as such:
> newString[i] = *(_BYTE *)(username + (tryharderLength ^ *(char *)(username + i)) % usernameLength);

As it can be seen it reorders the original string's characters and rebuilds a string.

With this knowledge we can create a keygen for this crackme as such:

	var username = RandomString(5);

	var tryHarderLength = 26;

	var serial = new StringBuilder();

	for (int i = 0; i < username.Length; i++)
	{
		var current = (int)username[i];

		var xored = username[(tryHarderLength ^ current) % username.Length];

		serial.Append(Convert.ToChar(xored));
	}            

	Console.WriteLine($"Username is: '{username}', password is: '{serial}'");

Example output is:
> Username is: '5g5ju', password is: '5555g'<br />
>  Enter Your Username : 5g5ju<br />
>  Password : 5555g<br />
> Good Password !<br />

We can see that is passes the check.

## Take aways

We could see, that this crackme was more difficult because of the packing involved. As the UPX packer is well documented packing algorithm it was easy to circumvent it, but with different packers this can get more and more difficult to do so. After the PE was unpacked the reversing was much easier to do so, since we could just check through the code of the crackme and see all of its hardcoded strings ect. Packers need to be unpacked in memory, so during runtime we can most of the time see the real PE in memory view and unpack it that way if there are no more defense mechanics involved.



## Credits

Crackmes.one and the community for providing fun challanges.