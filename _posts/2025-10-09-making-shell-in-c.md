---
title: "Building a simple Shell in C"
date: 2025-10-09
categories: [C, Systems]
tags: [shell, c, fork, exec, linux]
layout: post
description: "Step-by-step guide to implement a simple Unix-like shell in C."
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
- Friendly Interactive SHell (**fish**) — [fishshell.com](https://fishshell.com/)
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

> - If you don't call `wait()`, the child becomes a zombie process (terminated but not cleaned up).
> - Shell uses `wait()` to avoid zombies.
{: .prompt-info }

##### Q.) are all this wait() , ls , cd and all builtin commands ?
No not all are few very basic commands are only builtin like : cd , exit , fork() , wait() , that command prompt $ or > 

commands like ls , cat , grep , python , -l and all are external programs  these are stored in user/bin 

A way to check is to use `type`

![types command](/assets/images/types_command_example.png)

## Life cycle of an shell 

##### 1. Initialize 
Its just like as name suggest its initializes like read config files , set environment variables ( $PATH , $HOME etc ) , set internal Data structures ( shell does need memory to store variables aliases , paths , fancy colors etc )

##### 2. Interpret ( Main Loop )
this keeps it running till user exits it , its mainly parses the commands user types , keeps it active , keeps prompting that $ or > again and again 
creates child process for commands using `fork()`, also waits for child process to get complete.

##### 3. Terminate 
exits the shell , free memory and resources like release buffers , close open files, etc


## Planning the Mini Shell
we aren't making an crazy shell with plugins and configs and all we are making an MVP to learn concepts and all so we will keep things minimal, here we will make an basic plan of our shell what all it should have as as MVP , this of it as an overview 

- **Display prompt** : in this we are gonna make that $ or > sign or prompt like yourshell> , so that user know this is where he has to write the commands
- **Read inputs** : i think this is self explanatory because like if it cannot read the command then how will it executes it , normally we can use fgets() or readline() but we wont we will implement our own read_line() function for learning purposes.
- **Parse Input** : split the input commands and arguments for example `ls -l /home` here ls is the command and the -l, /home are the arguments so we need to split them and parses it
- **Handle Builtin Commands** : we already discussed there are two types of commands builtin and external , here we will focus on the builtin commands like cd , exit etc etc 
- **Handle external commands** : I think it is self explanatory

here below is the basic flow chart for the main loop 

![flowchart for Main loop](/assets/images/FLow_chart_mini_shell.png)

## Basic loop of a shell
now that we are talked a lot about basic stuffs and planning and all that now, lets get into actual coding part , firstly we will make an main loop for out shell 

we are making this loop so that during this loop we can handle any commands in 3 steps :
- Read 
- parse 
- Executes


```cpp
int main(int argc, char **argv)
{
    jeet_loop();
    return EXIT_SUCCESS;
}
```


Tis if like our main function its pretty basic , still let me break it down 

`int argc` is the count of arguments which will be passed , example if you did ls , the argc is 1 , but if you did `ls -l /home` then its 3 , think why ?
and now char \*\*argv its an double pointer , why double because argv is an arry of strings not char , if it was char then *argv would have been sufice

```
argv → [ptr0] [ptr1] [ptr2] [NULL]
        ↓      ↓      ↓
        "ls"   "-l"   "/home"
```


here is Memory Layout Visualization for `ls -l /home`


```
Memory Layout:
──────────────────────────────────────────────────────────
argv (address: 0x1000)
  │
  ├─► argv[0] (address: 0x2000) ──► ['l' 's' '\0']
  │
  ├─► argv[1] (address: 0x3000) ──► ['-' 'l' '\0']
  │
  ├─► argv[2] (address: 0x4000) ──► ['/' 'h' 'o' 'm' 'e' '\0']
  │
  └─► argv[3] = NULL
```


Now lets think why did we use `char **argv` only we could have also used `char *argv[]` and that an question for yourself to figure out.
( its same you can use it but still do once think)

And then we have our main jeet_loop() and return statement that is , now lets move onto the loop part 

#### Now lets discus the Main loop `jeet_loop` 
Loop is the heart of this mini shell becomes it is what keep running it whole like keeps prompting that $ or > , reading lines , parsing etc.
first here is the code for it then we will break it down in detail

```c
void jeet_loop(void)
{
  char *line;
  char **args;
  int status;

  do {
    printf("> ");
    line = jeet_read_line();
    args = jeet_split_line(line);
    status = jeet_execute(args);

    free(line);
    free(args);
  } while (status);
}
```

Here we are using an simple do while and making sure our shell keeps saying > or $ so that user knows shell is active , then we are simply reading the input of user via jeet_read_line and then tokiniszing it using jeet_split_line its like separating command string into program and arguments and then executing it , this loop is fairly simple and we are and then we are freeing the memory used via free() this step is very important , if you don't free it , it can cause memory leaks 

Example of memory leak , lets say you allocated 1MB and then never free it and it is inside an loop then it will keep adding 1MB each time thus consuming all memory and making system slower. hence its best practice to free the memory.


## Reading a line
here we are gonna make an function which reads line as the user inputs it we gonna make whole ourself for learning purposes we are not gonna use getline or fgets , so for learning purposes we gonna make our own.

So , it will be like we dont know how long is the command user is gonna type so we are first gonna allocate an memory block and if he exceeds that then we reallocate more memory to it 

```c 
char *read_jeet_line(void)
{
    int buffersize = JEET_RL_BUFFER;
    int postion = 0;
    char *buffer = malloc(sizeof(char) * buffersize);
    int c;

    if(!buffer){
        fprintf(stderr, "jeet says : Allocation error\n");
        exit(EXIT_FAILURE);
    }

    while(1){
        c = getchar(); 
        if(c == EOF || c == '\n'){
            buffer[postion] = '\0';
            return buffer;
        }
        else{
            buffer[postion] = c;
        }
        postion++;


        if(postion >= buffersize){
            buffersize += JEET_RL_BUFFER;
            buffer = realloc(buffer, buffersize);
            
            if(!buffer)
            {
                fprintf(stderr , "jeet says : Allocation error\n");
                exit(EXIT_FAILURE);
            }

        }
    }

}
```

we have an buffer as pointer which points to the memory where the text is getting stored , and the memory allocation is done by malloc , the postion is just for keeping track of charaters , then we have an infite loop where we are getting each charaters and why is c integer ?? its coz EOF is -1 so to check if we reached EOF ( end of file ) thus we are using int.

```cpp
if (!buffer) {
    fprintf(stderr, "Jeet: allocation error\n");
    exit(EXIT_FAILURE);
  }
```

what is this check ?? its like checking if system is not out of memory meaning no more memory to allocate and then buffer will be NULL and thus !NULL is true it will say allocation error and exit.


```cpp
// If we hit EOF, replace it with a null character and return.
    if (c == EOF || c == '\n') {
      buffer[position] = '\0';
      return buffer;
    } else {
      buffer[position] = c;
    }
    position++;


```

This part of code is for check EOF or new line , it is fairly simple , if it is EOF or \n then place an null terminator (\0) at the end of the sting which we have built , this is like an special charater which doesnt print but its an way to tell C that this is the end of sting

```cpp
// If we have exceeded the buffer, reallocate.
if (position >= bufsize) {
  bufsize += LSH_RL_BUFSIZE;
  buffer = realloc(buffer, bufsize);
  if (!buffer) {
    fprintf(stderr, "Jeet: allocation error\n");
    exit(EXIT_FAILURE);
  }
}
```

I previously mentioned that we don't know how long is the command so we need dynamic memory allocation thus we check if the Postion which exceeds the buffersize then we reallocate the memory via realloc , and after reallocation we again check if system is out of memory or not coz we just have reallocated more memory. now we will move on to the split line or parsing line function 


## Spliting line so we can get args

