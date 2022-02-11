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
>main		   push    rbp
>main+1  mov     rbp, rsp
>main+4    add     rsp, 0FFFFFFFFFFFFFF80h 
>main+8    call    __main          
>main+D    call    message      
>main+12   lea     rcx, Format     ; "Type your username: "
>main+19   call    printf         
>main+1E   lea     rax, [rbp+Destination]
>main+22   mov     rdx, rax
>main+25   lea     rcx, a20s       ; "%20s"
>main+2C   call    scanf       
>main+31   lea     rax, [rbp+Destination]
>main+35   mov     rcx, rax
>main+38   call    checkUsername  
>main+3D   mov     [rbp+var_4], eax
>main+40   cmp     [rbp+var_4], 0  
>main+44   jnz     short loc_401645 ;
>main+46   lea     rax, [rbp+Source] 
>main+4A   mov     rcx, 746569636F736640h
>main+54   mov     [rax], rcx
>main+57   mov     word ptr [rax+8], 79h 
>main+5D   lea     rdx, [rbp+Source] 
>main+61   lea     rax, [rbp+Destination] 
>main+65   mov     rcx, rax      
>main+68   call    strcat         
>main+6D   lea     rdx, [rbp+Destination] 
>main+71   lea     rax, [rbp+Source] 
>main+75   mov     rcx, rax      
>main+78   call    strcpy        
>main+7D   jmp     short loc_401671 

Checking through the assembly we can see, that there is a call to printf with a string literal parameter and a call to scanf which's return value is passed into a function called checkUsername. The parameter passing is done with
> mov rcx, rax

which moves the return value of scanf into rcx, that is used to pass parameters according to the x64 calling convetion.
Let's see checkUsername: 
>checkUsername      push    rbp
>checkUsername+1    mov     rbp, rsp
>checkUsername+4    sub     rsp, 30h     
>checkUsername+8    mov     [rbp+Str], rcx
>checkUsername+C    mov     rcx, [rbp+Str] 
>checkUsername+10   call    strlen          
>checkUsername+15   mov     [rbp+var_4], eax
>checkUsername+18   cmp     [rbp+var_4], 1  
>checkUsername+1C   jle     short loc_40157B 
>checkUsername+1E   cmp     [rbp+var_4], 7 
>checkUsername+22   jg      short loc_40157B 
>checkUsername+24   mov     eax, 0
>checkUsername+29   jmp     short loc_40158D 
>checkUsername+2B       loc_40157B:                             
>checkUsername+2B                                              
>checkUsername+2B   cmp     [rbp+var_4], 7 
>checkUsername+2F   jle     short loc_401588
>checkUsername+31   mov     eax, 1
>checkUsername+36   jmp     short loc_40158D
>checkUsername+38       loc_401588:                            
>checkUsername+38   mov     eax, 2
>checkUsername+3D
>checkUsername+3D       loc_40158D:                            
>checkUsername+3D                                              
>checkUsername+3D   add     rsp, 30h      
>checkUsername+41   pop     rbp
>checkUsername+42   retn                    
>checkUsername+42       checkUsername   endp

It does a call to strlen, then stores the return value in a local variable. Then checks to see if the length is 1 or less returns 2, if it's between 2 and 7 (inclusive) it returns 0 and if it's above 7 it returns 1.
Back in the main function it stores the return value of checkUsername and then performs a couple of checks. If the return value is not zero it jumps to 0x401645. If the return value is zero it load a value into the pointer stored in rax.
> main+4A   mov     rcx, 746569636F736640h
>main+54   mov     [rax], rcx
>main+57   mov     word ptr [rax+8], 79h 

This does seem like a hardcoded string literal, converting it to a character string it returns to be 'teicosf@' and then basically appends a 'y' at the beginning of the character string. Since, the stack grows from higher addresses to lower addresses, and rax is the beginning of the string, adding the 'y' to the address at rax+8 means adding it to the beginning of the string. So, it returns to be: 'yteicosf@'. Know this is encoded as little endian we can understand this reads as '@fsociety'. 
It then builds the input parameters for the strcat call using the x64 calling convetion, utilising rcx and rdx to pass the pointer values. Essentially, it takes our input that was entered as username and adds @fsociety to it. It saves this value in a local variable.

>main+AB       loc_401671:                         
>main+AB                                              
>main+AB   lea     rcx, aTypeYourPasswo ; "Type your password: "
>main+B2                    call    printf        
>main+B7   lea     rax, [rbp+Str2] 
>main+BB   mov     rdx, rax
>main+BE   lea     rcx, a30s       ; "%30s"
>main+C5   call    scanf         
>main+CA   lea     rdx, [rbp+Str2] 
>main+CE   lea     rax, [rbp+Source] 
>main+D2   mov     rcx, rax     
>main+D5   call    strcmp      
>main+DA   mov     [rbp+var_8], eax
>main+DD   cmp     [rbp+var_8], 0  
>main+E1   jnz     short loc_4016B7
>main+E3   lea     rcx, aHelloFriendYou ; "\nHello, friend. You successfully cr4ck"...
>main+EA   call    printf         
>main+EF   jmp     short loc_4016C3
>main+F1       loc_4016B7:                            
>main+F1   lea     rcx, aIMSorryYouAreN ; "\nI'm sorry. You are not supposed to be"...
>main+F8   call    printf       
>main+FD       loc_4016C3:                             
>main+FD   mov     eax, 0
>main+102  sub     rsp, 0FFFFFFFFFFFFFF80h
>main+106  pop     rbp
>main+107  retn                  
>main+107      main            endp

Then it jumps to 0x401671, which asks for the password, compares the return values and displays whether you authenticated successfully or not.
If the string compare at main+D5 returns anything, but zero we get the bad message, essentially, if you have entered the same thing, that is in the string, that was built before, you get the good message, otherwise you face the bad one. Keep a note of this.

Let's see the other branch of checkUsername's return value, when it's return value is 1, the jump to 0x401645.
>main+7F       loc_401645:                          
>main+7F   088                 cmp     [rbp+var_4], 1  
>main+83   088                 jnz     short loc_401667
>main+85   088                 lea     rax, [rbp+Source]
>main+89   088                 mov     dword ptr [rax], '.rM'
>main+8F   088                 lea     rdx, [rbp+Destination] 
>main+93   088                 lea     rax, [rbp+Source] 
>main+97   088                 mov     rcx, rax      
>main+9A   088                 call    strcat       
>main+9F   088                 jmp     short loc_401671 

We can see, that it concatenates our username with a prefix of 'Mr.'. Again since this is little endian, the string is stored backwards as a hexadecimal value.

Back to the string compare at main+D5. Essentially we can identify two ways. A username that is 2 to 7 long, we get the password of: 
><2 to 7 characters username>@fsociety

or

> Mr.<8 or more characters username>

And running the crackme it does indeed let us go through to the good message.

Let's see, if we can patch this exe to always get to the good message.

>main+D5   call    strcmp         
>main+DA   mov     [rbp+var_8], eax
>main+DD   cmp     [rbp+var_8], 0  
>main+E1   jnz     short loc_4016B7 
>main+E3   lea     rcx, aHelloFriendYou ; "\nHello, friend. You successfully cr4ck"...
>main+EA   call    printf        

We can see, that the string compare's value is used to jump if its not zero. If we wouldn't do the jump we could always reach the good message. So, noping the jump essentially modifies the flow of the code to never reach the bad message. After the patch the comparison code would look like this. We replace the 75 0E bytes to 90 90 to fill the exact amount of bytes that the jnz jump used with the byte code of nop.
>main+DD   cmp     [rbp+var_8], 0 
>main+E1   nop                   
>main+E2   nop                  
>main+E3                    lea     rcx, aHelloFriendYou ; "\nHello, friend. You successfully cr4ck"...
>main+EA                    call    printf       

## Credits

Crackmes.one and the community for providing fun challanges.