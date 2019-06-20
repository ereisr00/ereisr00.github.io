# 010Editor

010Editor is a hex-editor that includes a sophisticated template and scripting engine.
One of the major advantages of 010Editor is that it contains a large repository of templates that can be downloaded and used to parse new file formats. These templates are used to present an elaborate definition of a file and its structure in an easily digestable form (colors and windows). 

Just as have many other people, I've been using 010Editor for a while. This software is a leader in its specific market, and it's hard to find a good alternative. I was curious on how the 010Editor scripting engine was implemented, and whether the implementation was secure, so I (naturally) decided to audit the security of the scripting engine.

## Templates

The scripting engine can be used for running scripts or templates. Both of which run the same C-like code, but templates have additional restrictions (some functions require explicit user permission to execute). The code is compiled and executed by the scripting engine, and is manifested as a template file (.bt) or as a script file (.1sc). New templates can be uploaded to a central repository, from which other users can download and make use of new templates. At the time of writing, this repository contains a large number of templates that can be used to parse many different types of file formats. When a new file is opened, 010 checks if a template that can parse the file is found in the repository, if a template is found, 010Editor prompts the user with an option to download the template. Templates execute when a file associated with the template is loaded by the editor or when explicitly ran by the user.

## Attack surface

I started out by reviewing the documentation in order to gain an understanding of what kind of functionality the scripting engine provides the developer, and what kind of security guarantees the engine was supposed to provide. After looking at the list of functions provided by the engine, it was clear that the developers have implemented a large number of C-like functions. These functions are separated into 5 classes: Interface Functions, I/O functions, String functions, Math functions and Tool functions. 

010Editor also allows the use of external(DLL) functions, but the documentation clearly states that using these functions in templates requires explicit permission by the user. The usage of these functions in scripts is permitted. This situation made it clear that exploitation of a script file (.1sc) is not interesting, as scripts allow for the execution of arbitrary DLLs - and are expected to run at the discretion of the user (should be considered insecure).

Out of the five different classes of functions, one stood out - String functions. These functions deal with string and memory operations (strcpy, memcpy, memmove, printf etc...) - this kind of logic is known to be very bug-prone and difficult to implement securely. At that point, I figured that there's a fairly good chance that bugs would exist in the implementation of some of these functions, as they deal with the copying and moving of binary data (of which a string is made). As I am more interested in exploitable bugs (bugs that could result in code execution), the kind of memory corruptions that could occur from faulty implementation of these functions seemed like a good target (likely to overwrite memory).


## Reviewing the string functions

After reviewing a bunch of different functions, and finding some bugs that I deemed unexploitable, I came across the following (truncated and commented) code.

```Assembly

/*
 *  Internal implementation of function Memcpy in the template engine
 */
Memcpy(arg1, arg2, n, dest_offset, src_offset)
  ...
  mov     edi, [rdx+8]    // dest_offset
  mov     rdx, [rax+20h]
  mov     rax, [rax+10h]
  mov     ecx, [r9+4]
  mov     esi, [rdx+8]    // src_offset
  mov     edx, [r8+4]
  mov     eax, [rax+8]
  sub     ecx, edi        // dest_len -= dest_offset 1) no check on dest_offset for negative value
  sub     edx, esi        // src_len -= src_offset   2) no check on src_offset for negative value
  cmp     ecx, edx        // if (dest_len < src_len)
  cmovle  edx, ecx        //   copy_length = dest_len
  cmp     edx, eax        
  cmovg   edx, eax        
  test    edx, edx        
  jle     short loc_XXX 
  movsxd  rdi, edi        
  movsxd  rsi, esi        
  add     rdi, [r9+8]     // dest += dest_offset     3) passing dest += dest_offset could lead to underflow
  add     rsi, [r8+8]     // src += src_offset       4) passing src += src_offset could lead to underflow
  movsxd  rdx, edx        // n
  call    _memcpy         ; Call Procedure
...


```

As the comments specify, this is the internal implementation of the Memcpy function. 
This function has the following signature:
void Memcpy( uchar dest[], const uchar src[], int n, int destOffset=0, int srcOffset=0 )

The bug here is that the code does not validate destOffset and srcOffset properly.
These variables can be assigned a negative value, which can result in an underflow and corruption of the heap.

## Exploitation

At this point I had a memory corruption vulnerability, but I still needed something to overwrite. Surprisingly, the binary was not compiled with aslr (linux version) - which meant that if I could get a primitive to write an absolute address, I could easily overwrite the GOT and redirect execution.

As I already reversed the code, I knew that the string variable was pointed to by a structure that represented the scripting engine's "string object".
I realized that I could set the heap in such a way that this structure will be placed at a known location behind the array that I can underflow - this way I could overwrite this pointer to an absolute address of my choosing.
After that pointer is overwritten, the "string object" would point to wherever I want, and I could use it to read/write arbitrary memory.

To summarize, the exploitation follows these steps:

1. Initialize a string
2. Overwrite the pointer to the string with an address of a known function in the got
3. repeat 2 as much as needed to figure out which libc is running (by comparing the non-randomized bits to known libc addresses)
4. Calculate the address of system in libc
5. Initialize another string
6. Overwrite the pointer to a function in the got that we can trigger
7. Overwrite the got from bullet 6 with system
8. Trigger the function from bullet 6 with whatever command I want to run

## Closing

As 010Editor is a fairly popular product, these kinds of bugs can be used to attack a large number of people.
If a malicious template is uploaded to the central repository it could be used as an attack vector against anyone who downloads the template.
It's also possible to exploit a non-malicious template that uses a vulnerable function by sending a malicious file to a victim machine and inducing the victim to open that file with 010Editor (a less promising attack vector).

During the audit of 010 Editor version 9.0.1, numerous exploitable (and non exploitable) bugs were found.
After gathering up the findings and completing the POC, I contacted Sweetscape and presented them with the findings.
They quickly responded and cooperatively worked on fixing all the issues found. The bugs were fixed in version 9.0.2.

# Update

the following CVE IDs were assigned for the vulnerabilities:

memcpy_rce.bt: CVE-2019-12551
null_deref.bt: CVE-2019-12552
strcat_heap_overflow.bt: CVE-2019-12553
SubStr.bt: CVE-2019-12555
WSubStr.bt: CVE-2019-12554
