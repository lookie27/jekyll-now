---
layout: post
title: Getting started with Ghidra
---
# What is Ghidra
Ghidra is a Open Source decompiler released by the NSA. It is used to reverse engineer compiled programs. 

# Goals
* Get or feet wet this Ghidra
* Reverse engineer a compiled C program

# Prerequisites
* Download [Ghidra](https://ghidra-sre.org/)
* Download [Java JDK](https://www.oracle.com/java/technologies/javase-downloads.html)
* Download the [HelloWorld binary](https://github.com/lookie27/ghidra-hello-world/raw/master/HelloWorld)

# Getting started
Extract Ghidra and execute `ghidraRun.bat (Windows) 
						    ghidraRun.sh (Linux)`

You will be met with a screen that looks like
![]({{ https://github.com }}/lookie27/ghidra-hello-world/blob/master/Blog%20Resources/GhidraNewProject.png )

Click `File -> New Project`
Select `Non Shared Project`
Change the project directory to where you want the project to exist then give the project a name.

Now click `Code Browser` (dragon's head) under Tool Chest.


On the window that just opended, click `File -> Import File` and select the HelloWorld binary.

A import settings window will open and Ghidra will 'guess' the format and language. Ghidra should be able to figure out this binary, so just click OK.

If you get a popup that say HelloWorld has not been analyzed yet, click yes otherwise ato the top click `Analysis - > Auto Analyze`
Leave the default analyze options and click analyze.

You screen will look similar to 

On the left hand side, under Symbol Tree, open functions and click entry

Now on the right hand side of the scree, you should see a decompiled view of entry.

Now this is where decompilation gets harder.
Ghidra does its best with what it has, but we both know that is not what the C program I wrote looks like.

Lets pretty this up.

We know the entry point of a C program is main, so lets rename the entry function. Right click on entry and rename it to main.

We can also see that the return value of main is 0, but the function definition still says undefined8 or undefined type that is 8 bytes. We can assume that this is an int. Right click on main and edit function signature, then change the return type to int.

On line 8, we see [__strcpy_chk](https://refspecs.linuxbase.org/LSB_4.1.0/LSB-Core-generic/LSB-Core-generic/libc---strcpy-chk-1.html) and a quick google search tells us that that is strcpy with a buffer overflow check. __strcpy_chk(char* dest, char* src, size_t size) has the following signature. We can see that the size specified is 0xffffffffffffffff, so there is a good chance that strcpy was used. 

Lets change the method signature for __strcpy_chk to be that of int strcpy(char*, char*).

Now we can see that we are copying "HelloWorld!\n" into pcVar1. Lets change the name of pcVar1 to something more helpful. Right click on pcVar1 and rename to helloWorldString.

Now that we have a main function that looks pretty clean, lets see what it does
calls _mysteryFunction
mallocs 12 bytes to hellowWorldString
copys "helloWorld!\n" into helloWorldString
calls _printString with helloWorldString
frees hellowWorldString


lets dig into _printString, double click on it or open via the Symbol tree.

First thing we see is that __strcpy_chk has already been replace by strcpy.
Since we know we pass a string into _printString srename param_1 to stringValue.

We also see that sVar1 holds the strlen of stringValue, so change it to stringLength.

It looks like if stringValue has a lenght of 0, we put "Good night world!\n" into stringValue, then replace the first letter with a F and print it out. Otherwise we print stringValue.

Now lets look at _mysteryFunction, click on it in the Symbol Tree.

This might look confusing, because it is. What is uRam0000000000000000?!

Sometime, it helps to look at the assembly,

If we click on _mysteryFunction in the Decompile view, we can see that it light up in the Listing view. Now lets follow the assembly:
`
PUSH 	RBP   								Push the frame pointer (FP) to RSP
MOV 	RBP, RSP 							Move the FP into RBP
MOV 	qword ptr [RBP + _local_10], 0x0	Move the address of 0x0 into local variable on the stack
MOV		RAX,qword ptr [RBP + local_10]      Move the address of the local variable into RAX
MOV     byte ptr [RAX],0x5					Move 5 into the value at address define in RAX which is 0x0.
`

To sum that up, we create a local variable and set its address to 0x0 then set its value to 5 which would look like
`
void* p = 0;
p = 5;
`

And to summarize this whole thing, we have a program that segfaults.




