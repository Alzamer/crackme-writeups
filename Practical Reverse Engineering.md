# Practical Reverse Engineering Solutions
I'll paste my solutions to the exercises here
## Chapter 1: x86 & x64 Architecture
### Exercise 1, page 30
**01:** `mov edi, [ebp+8]` ; ebp is normally used as a base pointer - now it points to a value on the stack (argument).
**02:** `mov edx, edi` ; And now we move our address to edx
**03:** `xor eax, eax` ; Preferred way to set the register value to 0
**04:** `or ecx, 0FFFFFFFFh` ; Bitwise operation, we don't need to know the initial ecx value, because it ~~will contain 0xFFFFFFFF anyway~~ uh, wait, why are we trying to move 8-byte value into the 32-bit register? Ok, I've checked it in nasm, and it seems like we're just setting the ecx value to -1
**05:** `repne scasb` ; Repeat ecx times scasb instruction - but our ecx is equal to -1, and it is decrementing on each iteration. repne instruction "breaks" either when ecx = 0 or zf = 0 according to instruction reference, and zf is set when scasb instruction detects what is in eax (in our case it is just 0).
**06:** `add ecx, 2` ; ecx += 2 
**07:** `neg ecx` ; ecx = !ecx ;
**08:** `mov al, [ebp+0Ch]` ; Lower 8 bits of eax now point to another local variable
**09:** `mov edi, edx` ; We'll restore the value of edi to [ebp+8].
**10:** `rep stosb` ; and here's the second loop, running ecx times, copying the value from eax to edi byte by byte.
**11:** `mov eax, edx` ; The return value is usually stored in the eax register, and that's probably it.

#### Conclusion:
This is probably just a fragment of some function (we get it's local variables from ebp+8 or ebp+0Ch). In line 05 we have a loop (repne) that runs until it detects a null byte - so I'm guessing that [ebp+8] and [ebp+0Ch] are strings.

I think this function, from lines 01 to 07, determines the length of the string passed as an argument - I've tried this interesting formula from lines 05 to 07 on some sample strings, and it gives the correct string length every time - I didn't know that before.
OK, so our string length is stored in ecx, and in lines 08 to 10 we're simply overwriting the string [ebp+8] with a value stored in [ebp+0Ch].

### Exercise 1, page 36
`call` pushes the next instruction to the stack, and jumps to specified address
`ret` pops the return address from the stack and jumps there

Unfortunately, eip is a special register, and we can't get access to it's value using conventional methods (ie. `mov eax, eip`). However, there is a trick to get its value 

**01:** `get: mov eax, [esp]`
**02:** `ret`
**03:** `call get`

When we're calling get at line 03, we are pushing the address of the next instruction, and then we're jumping to get label - then we're dereferencing esp, which points at the top of the stack. So far, our stack contains only the value of the return code, which is equal to the eip value we had when the program was at 03 line.

### Exercise 2, page 36
eip is a register, which contains address of the next instruction to be executed. Of course `mov eax, eip` won't work, so we can use:

    jmp 0xAABBCCDD
we can also do
	  

    call 0xAABBCCDD

because call is a jump, which pushes the return address before jumping. Another way to do this

    push 0xAABBCCDD
    ret
