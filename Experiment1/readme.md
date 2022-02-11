# Reversing Experiment #1

In this page I will be doing some reverse engineering on a basic crackme. These are made with the purpose of being reversed, so there is no legal constraints regarding that. The one, that I will be reversing can be found at: https://crackmes.one/crackme/6194f35633c5d44c61906fe6



#### Note

This crackme was compiled with debug information, so it makes it pretty easy on the eye to decypher what is happening.

## Exploration

Fire up a vbox VM and run the executable to see how it acts:
> Type your username: test_username
> Type your password: test_password

It asks for a username and then waits for a password, then based on those inputs returns whether the pair passes the tests.
> I'm sorry. You are not supposed to be here.

Let's open the executable in a disassembler. We can see that the exe comes with function names, so we can just search for the main function to get to the C main() entry point. 
>main		   push    rbp											 <br />
>main+1  mov     rbp, rsp											 <br />
>main+4    add     rsp, 0FFFFFFFFFFFFFF80h 							 <br />
>main+8    call    __main          									 <br />
>main+D    call    message      									 <br />
>main+12   lea     rcx, Format     ; "Type your username: "			 <br />
>main+19   call    printf         									 <br />
>main+1E   lea     rax, [rbp+Destination]							 <br />
>main+22   mov     rdx, rax											 <br />
>main+25   lea     rcx, a20s       ; "%20s"							 <br />
>main+2C   call    scanf       										 <br />
>main+31   lea     rax, [rbp+Destination]							 <br />
>main+35   mov     rcx, rax											 <br />
>main+38   call    checkUsername  									 <br />
>main+3D   mov     [rbp+var_4], eax									 <br />
>main+40   cmp     [rbp+var_4], 0  									 <br />
>main+44   jnz     short loc_401645 ;								 <br />
>main+46   lea     rax, [rbp+Source] 								 <br />
>main+4A   mov     rcx, 746569636F736640h							 <br />
>main+54   mov     [rax], rcx										 <br />
>main+57   mov     word ptr [rax+8], 79h 							 <br />
>main+5D   lea     rdx, [rbp+Source] 								 <br />
>main+61   lea     rax, [rbp+Destination] 							 <br />
>main+65   mov     rcx, rax      									 <br />
>main+68   call    strcat         									 <br />
>main+6D   lea     rdx, [rbp+Destination] 							 <br />
>main+71   lea     rax, [rbp+Source] 								 <br />
>main+75   mov     rcx, rax      									 <br />
>main+78   call    strcpy        									 <br />
>main+7D   jmp     short loc_401671 								 <br />

Checking through the assembly we can see, that there is a call to printf with a string literal parameter and a call to scanf which's return value is passed into a function called checkUsername. The parameter passing is done with
> mov rcx, rax

which moves the return value of scanf into rcx, that is used to pass parameters according to the x64 calling convetion.
Let's see checkUsername: 
>checkUsername      push    rbp									   <br />
>checkUsername+1    mov     rbp, rsp							   <br />
>checkUsername+4    sub     rsp, 30h     						   <br />
>checkUsername+8    mov     [rbp+Str], rcx						   <br />
>checkUsername+C    mov     rcx, [rbp+Str] 						   <br />
>checkUsername+10   call    strlen          					   <br />
>checkUsername+15   mov     [rbp+var_4], eax					   <br />
>checkUsername+18   cmp     [rbp+var_4], 1  					   <br />
>checkUsername+1C   jle     short loc_40157B 					   <br />
>checkUsername+1E   cmp     [rbp+var_4], 7 						   <br />
>checkUsername+22   jg      short loc_40157B 					   <br />
>checkUsername+24   mov     eax, 0								   <br />
>checkUsername+29   jmp     short loc_40158D 					   <br />
>checkUsername+2B       loc_40157B:                                <br />
>checkUsername+2B                                              	   <br />
>checkUsername+2B   cmp     [rbp+var_4], 7 						   <br />
>checkUsername+2F   jle     short loc_401588					   <br />
>checkUsername+31   mov     eax, 1								   <br />
>checkUsername+36   jmp     short loc_40158D					   <br />
>checkUsername+38       loc_401588:                            	   <br />
>checkUsername+38   mov     eax, 2								   <br />
>checkUsername+3D												   <br />
>checkUsername+3D       loc_40158D:                            	   <br />
>checkUsername+3D                                              	   <br />
>checkUsername+3D   add     rsp, 30h      						   <br />
>checkUsername+41   pop     rbp									   <br />
>checkUsername+42   retn                    					   <br />
>checkUsername+42       checkUsername   endp					   <br />

It does a call to strlen, then stores the return value in a local variable. Then checks to see if the length is 1 or less returns 2, if it's between 2 and 7 (inclusive) it returns 0 and if it's above 7 it returns 1.
Back in the main function it stores the return value of checkUsername and then performs a couple of checks. If the return value is not zero it jumps to 0x401645. If the return value is zero it load a value into the pointer stored in rax.
> main+4A   mov     rcx, 746569636F736640h				  <br />
>main+54   mov     [rax], rcx							  <br />
>main+57   mov     word ptr [rax+8], 79h 				  <br />

This does seem like a hardcoded string literal, converting it to a character string it returns to be 'teicosf@' and then basically appends a 'y' at the beginning of the character string. Since, the stack grows from higher addresses to lower addresses, and rax is the beginning of the string, adding the 'y' to the address at rax+8 means adding it to the beginning of the string. So, it returns to be: 'yteicosf@'. Know this is encoded as little endian we can understand this reads as '@fsociety'. 
It then builds the input parameters for the strcat call using the x64 calling convetion, utilising rcx and rdx to pass the pointer values. Essentially, it takes our input that was entered as username and adds @fsociety to it. It saves this value in a local variable.

>main+AB       loc_401671:                         										  <br />
>main+AB                                              									  <br />
>main+AB   lea     rcx, aTypeYourPasswo ; "Type your password: "						  <br />
>main+B2                    call    printf        										  <br />
>main+B7   lea     rax, [rbp+Str2] 														  <br />
>main+BB   mov     rdx, rax																  <br />
>main+BE   lea     rcx, a30s       ; "%30s"												  <br />
>main+C5   call    scanf         														  <br />
>main+CA   lea     rdx, [rbp+Str2] 														  <br />
>main+CE   lea     rax, [rbp+Source] 													  <br />
>main+D2   mov     rcx, rax     														  <br />
>main+D5   call    strcmp      															  <br />
>main+DA   mov     [rbp+var_8], eax														  <br />
>main+DD   cmp     [rbp+var_8], 0  														  <br />
>main+E1   jnz     short loc_4016B7														  <br />
>main+E3   lea     rcx, aHelloFriendYou ; "\nHello, friend. You successfully cr4ck"...	  <br />
>main+EA   call    printf         														  <br />
>main+EF   jmp     short loc_4016C3														  <br />
>main+F1       loc_4016B7:                            									  <br />
>main+F1   lea     rcx, aIMSorryYouAreN ; "\nI'm sorry. You are not supposed to be"...	  <br />
>main+F8   call    printf       														  <br />
>main+FD       loc_4016C3:                             									  <br />
>main+FD   mov     eax, 0																  <br />
>main+102  sub     rsp, 0FFFFFFFFFFFFFF80h												  <br />
>main+106  pop     rbp																	  <br />
>main+107  retn                  														  <br />
>main+107      main            endp														  <br />

Then it jumps to 0x401671, which asks for the password, compares the return values and displays whether you authenticated successfully or not.
If the string compare at main+D5 returns anything, but zero we get the bad message, essentially, if you have entered the same thing, that is in the string, that was built before, you get the good message, otherwise you face the bad one. Keep a note of this.

Let's see the other branch of checkUsername's return value, when it's return value is 1, the jump to 0x401645.
>main+7F       loc_401645:                          										   <br />
>main+7F   088                 cmp     [rbp+var_4], 1  										   <br />
>main+83   088                 jnz     short loc_401667										   <br />
>main+85   088                 lea     rax, [rbp+Source]									   <br />
>main+89   088                 mov     dword ptr [rax], '.rM'								   <br />
>main+8F   088                 lea     rdx, [rbp+Destination] 								   <br />
>main+93   088                 lea     rax, [rbp+Source] 									   <br />
>main+97   088                 mov     rcx, rax      										   <br />
>main+9A   088                 call    strcat       										   <br />
>main+9F   088                 jmp     short loc_401671 									   <br />

We can see, that it concatenates our username with a prefix of 'Mr.'. Again since this is little endian, the string is stored backwards as a hexadecimal value.

Back to the string compare at main+D5. Essentially we can identify two ways. A username that is 2 to 7 long, we get the password of: 
><2 to 7 characters username>@fsociety

or

> Mr.<8 or more characters username>

And running the crackme it does indeed let us go through to the good message.

Let's see, if we can patch this exe to always get to the good message.

>main+D5   call    strcmp         																	   <br />
>main+DA   mov     [rbp+var_8], eax																	   <br />
>main+DD   cmp     [rbp+var_8], 0  																	   <br />
>main+E1   jnz     short loc_4016B7 																   <br />
>main+E3   lea     rcx, aHelloFriendYou ; "\nHello, friend. You successfully cr4ck"...				   <br />
>main+EA   call    printf        																	   <br />

We can see, that the string compare's value is used to jump if its not zero. If we wouldn't do the jump we could always reach the good message. So, noping the jump essentially modifies the flow of the code to never reach the bad message. After the patch the comparison code would look like this. We replace the 75 0E bytes to 90 90 to fill the exact amount of bytes that the jnz jump used with the byte code of nop.
>main+DD   cmp     [rbp+var_8], 0 																			  <br />
>main+E1   nop                   																			  <br />
>main+E2   nop                  																			  <br />
>main+E3                    lea     rcx, aHelloFriendYou ; "\nHello, friend. You successfully cr4ck"...		  <br />
>main+EA                    call    printf       															  <br />

## Credits

Crackmes.one and the community for providing fun challanges.