+++
title = 'Reverse Engineering fundamentals'
+++


These are all notes I took while learning reverse engineering (still a work in progress). I plan to complement them with further research, including videos and other resources. This is very uncharted territory for me so bear with me.

---

**RE theory**

### When you compile a binary in C , what happens ?

`gcc myProgram.c -o myProgram.exe`

It goes through these 4 stages , this is called `Language Processing System`.

(Preprocessor ---> Compiler ---> Assembler ---> Linker ) ----> myProgram.exe 

--

#### Now what happens when we run that program exe file ?

The program goes through a Loader. The loader will take the program and load it into main memory or   _**RAM**_.

Inside the RAM, there will be sections loaded by the loader inside the PE (portable executable file or exe) file. Common ones are `.text, .data, Heap, Stack.`

```
.text: has all the program (exe) file instructions. Basically all translated to the assembly code from high level C code.  

.data: stores Initialized and static data
			eg: int variableName = 10;

.bss (block starting symbol): stores uninitialized varibles . 
						eg: int vairableName; 

```

```
Heap: 
stores Dynamically allocated variables (allocated at runtime). 
it grows towards higher memory addresses , which means DOWNward visually speaking. 0x00000000
EXAMPLE: initialzing a array[], we dont know the size before hand until the user tells us how many numbers they want


Stack:
stores local variables, function parameters, return adresses and grows toward lower memory addresses, which means UPward visually speaking.
0x7fffffff
```


#### Okay, now Where does it go from the RAM?

It goes to the CPU (central processing Unit). Inside the CPU, the instructions from the RAM comes to the CU (control unit). From the CU , it goes to the ALU (arithmetic control unit) which executes instruction and sets *registers* and flags.

RAM ---> CPU ( [control unit ---> ALU ---> different registers, back and forth] )

Registers are very fast and ephemeral ( lasts very less ).

**Some Registers**:
	
```
+------------------------+ 64-bit → RAX
|        RAX             |
+------------+-----------+
             |
           EAX (bits 0–31) 32bit
             |
           AX  (bits 0–15) 16bit
           / \
         AH  AL (8 bits each)

This is the x86/x64 CPU register architecture. Basically the accumulator or any register can be named differently based on cpu architectures.

So when we are going to work with a 64bit exe file or a 32bit, expect different names for the registers.

Similarilly for the stack pointer:
	RSP, ESP or SP(8bits)

....and it goes on for every register.
```

*Accumulators* (rax,eax,ax): 
used for input/output, arithmetic and return values from functions.

*Base* (rbx, ebx, bx):
used for indexed addressing

*Counter* (rcx, ecx, cx):
stores loop count variables in iterative operations. eg for loops/while loops

*Data* (rdx, edx, dx):
input/output operations and sometimes extends rax for multiply /divide

*Stack Pointer* (rsp, esp, sp):
stores current position within the stack

*Base Pointer* (rbp, ebp, bp):
helps in referencing parameter and other stack variables by providing a fixed point in the stack, so they can be accessed using simple offsets. Holds the base of our current stack frame

*Instruction Pointer* (rip,eip,ip):
stores next instruction to be executed

*Source and Destination registers* (rsi/esi , rdi/edi ):
used as a source and destination indexes for string operations 

*flag registers*:
CF (carry flag) , ZF(zero flag), TF (trap flag)

These can be nicely visualized in `x64 debugger`, a fantastic RE (debugging) tool.


**some Common x86  Instructions**:

Read from right to left. Forget your everyday English reading convention.
```
`mov eax, ebx`    copies the value in ebx to eax .  

`mov eax, [0x70000000]`   move whatever value is in the memory address `[]` to eax. Brackets means "go-to-that-address first then load that data"

`nop`   No operation and do absolutely nothing,

`lea eax, [ebx+esi*4]`   load effective address, similar to move but it loads ebx+esi*4 instead of the data of that address like we saw at the bracketed [] address do. 

xor eax, eax;or eax, ebx
they take 2 registers , compare them, and stores the result in a flag register.

There are many examples but i am not goin to note it down.
 etc ........
```


*Stack operations:* 
....also instructions 

`push eax; pop ebx`

`push` means putting values to the top of the stack frame and `pop` means , removing them from the top of the stack.


*Call instructions:*
`call 0x41400000`    (sometimes we might see the function name here instead of hex value.)

calls a function; moves the value of EIP to stack and sets EIP to the start of the function at 0x41400000.  So that it knows what to do after finishing that function, i.e. , starting the next instruction.

`ret`   Pops the return address off of the stack into EIP.

All these cluster of *info* happens in the CPU

---

Day 2.
### Inner Workings of the Stack

~~~
^
I    0x00000000   Lower level addresses        (ESP)
I
I
I
I
I    0x7fffffff      Higher level addresses         (EBP)
I
~~~
The stack grows UP, to lower level addresses. When we say pop something of from the stack, it always means remove a item from the TOP of the stack.

To solidify the concepts here , do some practice CrackMEs.

Also learning how to use  *x64 debugger* helps big time as you can see registers , function calls on the stack and much more.

**No more notes now atm. Will add more when I understand more or might not add here at all.**

---

