
# Reversing Experiment #2

In this page I will be doing some reverse engineering on a crackme, that is indicated as difficulty 2. These are made with the purpose of being reversed, so there is no legal constraints regarding that. The one, that I will be reversing can be found at: https://crackmes.one/crackme/617581d833c5d4329c3452ce



## Exploration

Let's see what the executeable does when ran:
> user: test_username
> serial: test_serial

It asks for a username and then prompts for the serial, then based on those inputs returns whether the pair passes the tests. We get the bad message in this case. (Copied as is.)
> you are not a master hacking

Let's open the executable in a disassembler. What we can do is look at the Exports of the PE file. It lists a few TLSCallbacks and a main startup function.
>Name	Address <br />
>TlsCallback_0	000000000040EB90	<br />
>TlsCallback_1	000000000040EB60	<br />
>TlsCallback_2	000000000041B740	<br />
>mainCRTStartup	0000000000401500<br />

The mainCRTStartup seems to be a good place to start. It calls the _tmainCRTStartup function.
> call    __tmainCRTStartup <br />

Which then, does all sorts of setups, and at the end calls the main() function with argv and envp input parameters which seems to be a C style main entry function. This seems to be a place to look around. And indeed, if we open the this function, we can see hard coded strings which we saw during runtime, and scrolling through the function we can see the good and bad messages as well.
>Eg: <br />
>main+155  078                 lea     rdx, aGoodYouAreAMas ; "good you are a master hacking, but make"... <br />
>main+185  078                 lea     rdx, aYouAreNotAMast ; "you are not a master hacking" <br />

Let's quickly discuss, in case with hardcoded strings (not stack strings ect.), you can also search the strings coded into the binary. Most disassemblers come with string lookup features. If I open the string view for my disassembler, I will see the strings.
> .rdata:0000000000488018	00000034	C	good you are a master hacking, but make a keygen :/ <br />

Double clicking that takes me to the rdata section, on which I can look for references:
> Up	o	main+155	lea     rdx, aGoodYouAreAMas; "good you are a master hacking, but make"...<br />

And as it can be seen, it takes me to the same function.

Let's see what the function does and maybe without reversing the whole thing, we can figure out what is happening:
>main+45   078                 lea     rdx, Str        ; "user: "<br />
>main+4C   078                 mov     rcx, cs:_refptr__ZSt4cout ; std::ostream *<br />

Sets up an output of "user:" using C++'s << operator for strings and the reads in the input to a variable. Then, there are some jumps involved, it seems to be a loop, also there are calls to std::string::length and some comparison.
>main+7E   078                 mov     [rbp-10h+var_14], 0<br />
>main+85   078                 jmp     short loc_4015FD ; Jump<br />
...<br />
>main+D3   078                 lea     rax, [rbp-10h+ark_enteredUsername] ; Load Effective Address<br />
>main+D7   078                 mov     rcx, rax        ; this<br />
>main+DA   078                 call    _ZNKSs6lengthEv ; std::string::length(void)<br />

It seems it is iterating through our entered username and does some calculations on it. Which is a behaviour, that you would expect with serial keys.

In the loop body we see, that it is indeed looping through the string and after that xor's the value of the current char of the string and than adds the length of the string to that value, which then gets appended to a string. It might be building the serial key this way...
> main+94   078                 call    _ZNSsixEy       ; std::string::operator[](ulong long)<br />
> main+99   078                 movzx   eax, byte ptr [rax] ; Move with Zero-Extend<br />
> main+9C   078                 mov     [rbp-10h+ark_currentChar], al<br />
> main+9F   078                 movzx   eax, [rbp-10h+ark_currentChar] ; Move with Zero-Extend<br />
> main+A3   078                 xor     eax, 4          ; Logical Exclusive OR<br />
> ...<br />
> main+A8   078                 lea     rax, [rbp-10h+ark_enteredUsername] ; Load Effective Address<br />
>main+AC   078                 mov     rcx, rax        ; this<br />
>main+AF   078                 call    _ZNKSs6lengthEv ; std::string::length(void)<br />
>main+B4   078                 add     eax, ebx        ; Add<br />
>...<br />
>main+BD   078                 lea     rax, [rbp-10h+ark_calculatedSerial] ; Load Effective Address<br />
>main+C1   078                 mov     rcx, rax        ; this<br />
>main+C4   078                 call    _ZNSspLEc       ; std::string::operator+=(char)<br />

Let's see if we can test this theory. If I enter 'a' as the username it's representation in hex will be 61h. If I xor that with 4 and add the length of the username string to it we get 66h. We can convert that to ASCII and the result will be 'f'. So for the serial I enter 'f' and the crackme indeed greets me with the good message: 
> good you are a master hacking, but make a keygen :/<br />

So, knowing all this we can actually write a keygen for this, as it prompts us to do.

Here is a quick prototype I put together in C#. It's pretty straight forward, we generate a random string of length 3, then do the calculation we found from reversing.

    var username = RandomString(3);

    var serial = new StringBuilder();

    for (int i = 0; i < username.Length; i++)
    {
        var first = (int)username[i];

        var xored = first ^ 4;

        xored += username.Length;

        serial.Append(Convert.ToChar(xored));
    }

    Console.WriteLine($"Username is: '{username}', serial is: '{serial}'");

An example output is:
> Username is: '5PA', serial is: '4WH'

 Let's see, if we can also patch this file. Looking at the function we can see that we do a jump if zero call to a short location. (Basically means if the two strings didn't match we jump to the bad message.) So, patching this jz short to nops will always guide our code flow to the good message, which is what we want. 
For address 0x401683 we change the first two bytes 74 30 which is for the short jz to 90 90 which is two nops to fill in the exact amount of bytes we are patching. The result will be like this:
>main+151  078                 test    al, al          ; Logical Compare<br />
>main+153  078                 nop                     ; No Operation<br />
>main+154  078                 nop                     ; No Operation<br />
>main+155  078                 lea     rdx, aGoodYouAreAMas ; "good you are a master hacking, but make"...<br />

When we run the program we can enter any user and serial combination we will always get the good message:
>user: aawdawd<br />
>serial: awd<br />
>good you are a master hacking, but make a keygen :/<br />


## Credits

Crackmes.one and the community for providing fun challanges.