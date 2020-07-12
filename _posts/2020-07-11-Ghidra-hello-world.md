---
layout: post
title: Getting started with Ghidra
---
# What is Ghidra
Ghidra is a Open Source decompiler released by the NSA. It is used to reverse engineer compiled programs. 

# Goals
* Get our feet wet with Ghidra
* Reverse engineer a compiled C program

# Prerequisites
* Download [Ghidra](https://ghidra-sre.org/)
* Download [Java JDK](https://www.oracle.com/java/technologies/javase-downloads.html)
* Download the [HelloWorld binary](https://github.com/lookie27/ghidra-hello-world/raw/master/HelloWorld)

# Getting started

## Lauching Ghidra 
Extract Ghidra and enter it.  
If you are on Windows, execute runGhidra.bat  
If you are on Linux, execute runGhidra.sh  

This will launch Ghidra and you will be met with a screen that looks like  
![](https://raw.githubusercontent.com/lookie27/ghidra-hello-world/master/Blog%20Resources/GhidraNewProject.png)  
This is where you can manage your projects; Since this is our first project, we are going to create a new one.  

## Creating a new project

Click **File -> New Project**  
Select **Non Shared Project** (Because we will be the only ones working on it)  
Change the project directory to where you want the project to exist then give the project a name.  

It's that _simple_! 

Enough small talk, lets do some decompiling!

## Decompiling 

### Importing a binary

To decompile a binary, we need to load it into Ghidra. 

Click **Code Browser** (dragon's head) under Tool Chest, this will open the view where we will spend most of our time. 


On the window that just opended, click **File -> Import File** and select the HelloWorld binary.

A import settings window will open and Ghidra will attempt to figure out the format and language. Ghidra might assume the incorrect attributes and you may need to select them yourself, but in this case it is correct. 

### Binary Analysis 

Before you continue let's take a moment to talk about what importing did and what analyzing will do. 
Importing a file means that the machine code (10001010) of the imported file is loaded into Ghidra and can be read by it. 
The machine code is not very human friendly, so when we analyze it, Ghidra translates it into assembly. Now with the translated assembly, Ghidra will attempt to translate that into the high level language that it was written it. This is a very hard task and Ghidra uses a series of methods to construct a shell of the high level language. 

What do I mean by shell? Well, depending on how much information is left in the binary i.e: function names, function return types, variable names, variable types, ect. they may not exist and Ghidra can only tell you what it knows. So if a return type was an int, but the binary doesn't contain that information, Ghidra will tell you the return type is a 8 byte type. 

With that being said, let's continue.

If you get a popup that say HelloWorld has not been analyzed yet, click yes otherwise at the top of the page click **Analysis - > Auto Analyze**. 
Leave the default analyze options and click analyze.

You screen will look similar to 
![](https://raw.githubusercontent.com/lookie27/ghidra-hello-world/master/Blog%20Resources/DecompileView.png)

### Reconstruction

Earlier I said that Ghidra provides us with a shell, it is our job to fill that shell. 

We will use our **blood**, **sweat**, and **tears**... and maybe a little **sheer willpower** to reconstruct this program! 

On the left hand side, under **Symbol Tree** open functions and click **entry**.

Now on the right hand side of the screen, you should see a decompiled view of entry. 
![](https://raw.githubusercontent.com/lookie27/ghidra-hello-world/master/Blog%20Resources/MainDecompile.png)

This is where decompilation gets hard. Ghidra does its best, but we both know that doesn't look like C.

Lets pretty this up!

We know the entry point of a C program is main, so let's rename the entry function. **Right click** on entry and rename it to main.

We can also see that the return value of main is 0, but the function definition says undefined8 or undefined type that is 8 bytes. 

We can assume that this is an int. Right click on main and **edit function signature**, then change the return type to int.

On line 8, we see [\_\_strcpy\_chk](https://refspecs.linuxbase.org/LSB_4.1.0/LSB-Core-generic/LSB-Core-generic/libc---strcpy-chk-1.html) and a quick google search tells us that is strcpy with a buffer overflow check. 

 __strcpy_chk(char* dest, char* src, size_t size) has the following signature. We can see that the size specified is 0xffffffffffffffff, so there is a good chance that strcpy was used. 

Change the method signature for \_\_strcpy\_chk to be that of int strcpy(char*, char*). 

Now we can see that we are copying "HelloWorld!\n" into **pcVar1**. Right click on pcVar1 and **rename** to something more helpful, like **helloWorldString**.

That's much better! 

![](https://raw.githubusercontent.com/lookie27/ghidra-hello-world/master/Blog%20Resources/MainClean.png)

Now that we have a main function that looks pretty clean, lets see what it does 
```
calls _mysteryFunction 
mallocs 12 bytes to hellowWorldString 
copys "helloWorld!\n" into helloWorldString 
calls _printString with helloWorldString 
frees hellowWorldString 
```

Let's dig into **\_printString**, double click on it or open via the Symbol tree.

![](https://raw.githubusercontent.com/lookie27/ghidra-hello-world/master/Blog%20Resources/PrintDecompile.png)

First thing we see is that **\_\_strcpy\_chk** has already been replace by strcpy.

Since we know we pass a string into \_printString rename **param_1** to stringValue.

We also see that **sVar1** holds the strlen of stringValue, so change it to **stringLength**.

![](https://raw.githubusercontent.com/lookie27/ghidra-hello-world/master/Blog%20Resources/PrintClean.png)

It looks like if stringValue has a length of 0, we put "Good night world!\n" into stringValue and replace the first letter with a F then print it out. Otherwise we print out stringValue. 

Now lets look at **\_mysteryFunction**, click on it in the Symbol Tree. 

![](https://raw.githubusercontent.com/lookie27/ghidra-hello-world/master/Blog%20Resources/MysteryFunction.png)

This might look confusing, because it is! 

**What is uRam0000000000000000?!** 

Sometimes, it's helpful to look at the assembly. 

If we click on **\_mysteryFunction** in the Decompile view, we can see that it light up in the Listing view.

We will go through the assembly and figure out what is happening.  
```
PUSH 	RBP   								Push the frame pointer (FP) to RSP 
MOV 	RBP, RSP 							Move the FP into RBP  
MOV 	qword ptr [RBP + _local_10], 0x0	Move the address of 0x0 into local variable on the stack 
MOV		RAX,qword ptr [RBP + local_10]      Move the address of the local variable into RAX 
MOV     byte ptr [RAX],0x5					Move 5 into the value at address define in RAX which is 0x0. 
```
  
To sum that up, we create a local variable and set its address to 0x0 then set its value to 5 which would look like  
```
void* p = 0; 
p = 5; 
```  

Now that we have our decompiled C program we can put it all together to get  

```
void _printString(char *stringValue)
{
  size_t stringLength;
  
  stringLength = _strlen(stringValue);
  if (stringLength == 0) {
    strcpy(stringValue,"Good night world!\n");
    *stringValue = 'F';
    _printf("%s",stringValue);
  }
  else {
    _printf("%s",stringValue);
  }
  return;
}

void _mysteryFunction(void)
{
  void* p = 0x0;
  p = 5;
  return;
}

int main(void)
{
  char *helloWorldString;
  
  _mysteryFunction();
  helloWorldString = (char *)_malloc(0xc);
  strcpy(helloWorldString,"HelloWorld!\n");
  _printString(helloWorldString);
  _free(helloWorldString);
  return 0;
}

```

And our [actual program](https://github.com/lookie27/ghidra-hello-world/blob/master/HelloWorld.c) looked like

```
#include "stdio.h"
#include "stdlib.h"
#include "string.h"

void mysteryFunction()
{
    char* p = 0;
    *p = 5;
}

//prints string
void printString(char* strValue) 
{
    if (strlen(strValue) > 0) 
    {
        printf("%s", strValue);
    }
    else 
    {
        strcpy(strValue, "Good night world!\n");
        strValue[0] = 'F';
        printf("%s", strValue);
    }
}

int main(int argc, char *argv[]) 
{
    mysteryFunction();
    char* helloWorldString = malloc(12);
    strcpy(helloWorldString, "HelloWorld!\n");
    printString(helloWorldString);
    free(helloWorldString);
}

```
It wasn't an exact match, but it's good enough for Government work.

