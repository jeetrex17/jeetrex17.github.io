---
title: "Building a simple Shell in C"
date: 2025-10-09
categories: [C, Systems]
tags: [shell, c, parsing, fork, exec, linux]
layout: post
description: "Step-by-step guide to implement a simple Unix-like shell in C covering parsing, process control, pipes and redirection."
---
In this i am learning how to make an mini shell using c , i have followed this [article](https://brennan.io/2015/01/16/write-a-shell-in-c/) by [Stephen Brennan](https://brennan.io/) 

first of all i don't know anything about making an shell i have read few blogs , watched few videos and then here i am sharing what i learnt. and yah most of the code will be same as in article by Stephen Brennan 

## What exactly is shell ?

shell is an program which acts as an middle men between your OS and kernel , now that we know this one more question comes in mind what is an Kernel ? 

**kernel** is like an very core part of the OS which is like an main control center of your entire computer system , its like an bridge between CPU , RAM , disk etc etc to your Software

So , its like you enter commands in Shell , and the Kernel actually executes those commands 

making all work is done by kernel , shell is just an interface.

there are diff types of shells here are few examples :
- Bourne Again SHell (**Bash**) — [gnu.org](https://www.gnu.org/software/bash/)
- Z shell (**Zsh**) —  [zsh.org](https://www.zsh.org/)
- Friendly Interactive SHell (**fish**) — [fishshell.com+1](https://fishshell.com/)
- Korn Shell (**ksh**) — [kornshell.com](https://kornshell.com/)
- C Shell / TENEX C Shell (**csh / tcsh**) — [tcsh.org](https://www.tcsh.org/)
- PowerShell — [learn.microsoft.com](https://learn.microsoft.com/en-us/powershell/)


## How does shell even communicate to kernel ?

Answer is **"System calls"**. 
now questions come what is even an system call ? 
think this way system calls are like special request from user program to kernel  

user programs means any app or browser or your chatgpt copied python script for web scrapping ( btw this blog is not written by gpt )

So because user programs cannot directly communicate to hardware , we use shell that's it

Wait but why can user program directly access hardware ? simple reason for security reasons and stability as well there is no control guy to mange how much resource to give to what and all 

### Mechanism of communication 

here are few system calls used by shells :
- `fork()` → create a new process
- `exec()` → replace the current process image with a new program
- `wait()` → wait for a process to finish
- `read()` / `write()` → interact with input/output (like files or terminal)
- `open()` / `close()` → manage files

Lets say you wrote `ls -l` 
1. shell firsts separates ls and -l 
2. then it calls `fork()` its like making an new copy of shell itself ( creating an child process )
3.  now it replaces the Child process with that `ls` program and kernel loads `ls` into the memory
4.  we already know kernel is the guy who actually executes the program so it executes the `ls` program 
5. shell waits using `wait()` until ls is finished and then once finished it exits 
6.  shell continues and prompts for next command 

### Few questions one might have 
##### Q.) why even use `fork()` and make an child process ?
its because lets say it didn't make child process and ran that `ls` command it self then there wont be the next prompt right , so its very inconvenient , 
So , by calling `fork()` the shell creates a child process and that process runs that command and the main parent shell is still alive and can prompt for next command 

##### Q.) What happens after `wait()` is over? Does it kill the child process? and why even use `wait()`
Once `wait()` is over it it doesn't kill the chill process , `wait()` simply pauses the shell until the child process exits naturally (finishes execution).
After the child process is finished the kernel itself does the following 
- Kernel cleans up the child’s resources (this is called **reaping the process**).
- Shell gets the child’s **exit status** (success/failure).
- Shell continues and shows a prompt.

> [!NOTE]
> - If you don’t call `wait()`, the child becomes a **zombie process** (terminated but not cleaned up).
> - Shell uses `wait()` to avoid zombies.

##### Q.) are all this wait() , ls , cd and all builtin commands ?
No not all are few very basic commands are only builtin like : cd , exit , fork() , wait() , that command prompt $ or > 

commands like ls , cat , grep , python , -l and all are external programs  these are stored in user/bin 

A way to check is to use `type`

![types command](/assets/images/types_command_example.png)

## Life cycle of an shell 

#### 1. Initialize 
Its just like as name suggest its initializes like read config files , set environment variables ( $PATH , $HOME etc ) , set internal Data structures ( shell does need memory to store variables aliases , paths , fancy colors etc )

#### 2. Interpret ( Main Loop )
this keeps it running till user exits it , its mainly parses the commands user types , keeps it active , keeps prompting that $ or > again and again 
creates child process for commands using `fork()`, also waits for child process to get complete.

#### 3. Terminate 
exits the shell , free memory and resources like release buffers , close open files, etc 
