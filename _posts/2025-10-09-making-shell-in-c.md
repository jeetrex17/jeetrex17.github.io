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

```c
if (!buffer) {
    fprintf(stderr, "Jeet: allocation error\n");
    exit(EXIT_FAILURE);
  }
```

what is this check ?? its like checking if system is not out of memory meaning no more memory to allocate and then buffer will be NULL and thus !NULL is true it will say allocation error and exit.


```c
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

```c
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
as we are just making an mini shell so we will be making it simple only , we'll just use whitespace (spaces, tabs, newlines) as our separators. We won't worry about quotes or backslashes. so basically our this function will tokinise commands like `ls -l /home` into "ls" , "-l" , "/home" and then end of list . but how will we know the separators ? like from where to separate and for that we predefine them as #define LSH_TOK_DELIM " \t\r\n\a".

This code is fairly similar to the readline coz its kinda same logic like allocating memory for an array of pointers (here tokens) , and have an array of pointers (our tokens variable) that will act as a list of strings , allocate memory block to token , check if there is no allocation error. 

```c 
char **jeet_split_line(char *line){
    int buffersize = JEET_TOK_BUFFER;
    int postion = 0;

    char **tokens = malloc(sizeof(char *) * buffersize);
    char *token;

    if(!tokens){
        fprintf(stderr, "jeet says : Allocation error\n");
        exit(EXIT_FAILURE);
    }

    token = strtok(line , JEET_TOK_DELIM);
    while(token != NULL){
        tokens[postion] = token;
        postion++;

        if(postion >= buffersize)
        {
            buffersize += JEET_TOK_BUFFER;
            tokens = realloc(tokens, buffersize*sizeof(char *));

            if(!tokens){
                fprintf(stderr , "jeet says Allocation error\n");
                exit(EXIT_FAILURE);
            }
        }

        token = strtok(NULL , JEET_TOK_DELIM);   // contuning from where it left off

    }

    tokens[postion] = NULL;

    return tokens;
}
```


Here we have used `strtok()` and given line as its input what it will do it , it will find the first token and skiping any delimiters at the start. It returns a pointer to the beginning of that token (e.g., the 'l' in "ls"). there is an note to take that `strtok()` is destructive. It actually modifies your original line string. It finds the space after "ls" and replaces it with a null terminator (\0). This is how it "chops" the string. then we have an classic loop to keep tokiniszing until `strtok()` returns NULL , you may wonder like why in first we gave line as input in `strtok()` and then we gave NULL , its coz strtok only give one token at a time and then we pass NULL in an loop to loop through whole string.


## Launching Processes: The Heart of Our Shell
Okay, we've read a line, we've split it into arguments now what? Now we get to the real job of a shell that is actually running the command.
This part is the "magic" of how shells work, and it's all built on a three-step process.

-`fork()` - You clone your process.
-`exec()` - The clone (child) replaces itself with the new program.
-`wait()` - The original (parent) waits for the clone to finish.

```c
int jeet_launch(char **args){
    pid_t pid , wpid; // if pid = 0 its running child process , if pid > 0 then parrent else error 
    int status;

    pid = fork();
    if(pid == 0){   //child process 
        if (execvp(args[0], args) == -1){
            perror("jeet");
        }
        exit(EXIT_FAILURE);
    }
    else if(pid < 0){
        // error while forking
        perror("jeet");
    }
    else{
        do {
            wpid = waitpid(pid , &status, WUNTRACED);
        } while(!WIFEXITED(status) && !WIFSIGNALED(status));
    }

    return 1;
}
```

### 1. `fork()` cloning the parent
First, we need to run a new command (like ls) without killing our shell. If our shell just ran ls, our shell program would be gone! the solution is simple use `**fork()**` , The operating system creates an exact duplicate of our shell process. The original process is called **parent** and the duplicate one is called the **child** process and we will run the commands in the child process and to do that we will use `**exec()**`

### 2. `exec()` Replacing the child process
after calling `fork()` the child process is running the shell as well but we don't it to run shell we want to run the command which the user wants. So, the child process uses exec(). This function does one thing that is it replaces the entire current process with a new program.
The most important thing to know about exec is that **it never returns**

### 3. `wait()` wait till exec is over 
we dont want to spam the user with the > or $ sign coz the parent process is also running as well the child ( where the command will run ) so we need to use wait till child process is completed , or this, we use `waitpid()`. This tells the OS, "Hey, pause my (the parent's) execution. Just let me know when the child process (PID pid) finishes its job.

| `pid` Value | Who Am I? | What Does It Mean? | My Job Is To... |
| :--- | :--- | :--- | :--- |
| `pid == 0` | **Child** | "You are the newly created process." | `exec()` (run the new command) |
| `pid > 0` | **Parent** | "A child was created. Its ID is `pid`." | `wait()` (wait for the child to finish) |
| `pid < 0` | **Parent** | "Error: No child was created." | `perror()` (report the error) |


`execvp()`: Why execvp? The exec family has many functions.
The 'v' (execv...) means it takes an array (a vector) of arguments, which is perfect for our args array.
The 'p' (exec...p) means it will search the system PATH for the command. This lets us type ls instead of having to know the full path /bin/ls.

The `waitpid()` Loop: Why a do...while loop? We want to wait until the child is truly finished. A process can be stopped (like with Ctrl+Z) and then continued later. This loop ensures we only stop waiting when the child has exited (WIFEXITED) or been signaled to terminate (WIFSIGNALED)

## builtin commands
We have already learnt what builtin and external commands, builtin commands don't run in the child process inshort they don't follow that fork exec. Builtins are commands that the shell runs itself, inside its own process, without using `fork()` , and why is that ? lets understand it with an simple example of `ls` and `cd` , when we do `cd` we want to change our working directory and ls is just list the files , so lets say we use that fork exec way for `cd` then it will run in the child process and once child process and nothing happens in the parent process ( our shell ) so basically the working directory will remain unchanged. so i think you can get why we need builtin commands

```c 
// making bultin functions

int jeet_cd(char **args);
int jeet_help(char **args);
int jeet_exit(char **args);

char *builtin_str[] = {
    "cd", "help" , "exit"
};

int (*builtin_func[])(char **) = {
    &jeet_cd,
    &jeet_help,
    &jeet_exit
};


int jeet_num_builtin(){
    return (sizeof(builtin_func) / sizeof(char *));
}


// now implementing the builting func cd help and exit 

int jeet_cd(char **args){
    if(args[1] == NULL){
        fprintwf(stderr , "Taking u back to HOME\n");
        

        if(chdir(getenv("HOME")) != 0){
            perror("jeet");
        }
    }
    else {
        if(chdir(args[1]) != 0){
            perror("jeet\n");
        }
    }

    return 1;
}

int jeet_help(char **args) {
    int n = jeet_num_builtin();
    printf(
        "    ____. _________ ___ ______________.____    .____     \n"
        "   |    |/   _____//   |   \\_   _____/|    |   |    |    \n"
        "   |    |\\_____  \\/    ~    \\    __)_ |    |   |    |    \n"
        " /\\__|    |/        \\    Y    /        \\|    |___|    |___ \n"
        " \\________/_______  /\\___|_  /_______  /|_______ \\_______ \\\n"
        "                  \\/       \\/        \\/         \\/       \\/\n"
        "\n"
        "         Welcome to J shell\n"
        "------------------------------------------------------\n"
        "Built-ins: %d\n"
        "  cd    : change directory\n"
        "  help  : show this message\n"
        "  exit  : exit the shell\n"
        "\n",
        n
    );

    return 1;
}

int jeet_exit(char **args){
    printf("Bye - ciao - vale - αντίο - tchau - au revoir\n");
    return 0;
}
```

Now , we need to make an loop up table for the builtin function so like when an user types an commands and it checks if its an builtin commands or not , we can use an if else block if we want like this :

```c 
if (strcmp(args[0], "cd") == 0) {
  return jeet_cd(args);
} else if (strcmp(args[0], "help") == 0) {
  return jeet_help(args);
} else if (strcmp(args[0], "exit") == 0) {
  return jeet_exit(args);
} else {
  return jeet_launch(args);
}
``` 


this will most definatly work but its just hard to maintain and looks bad , every time you make an new builtin function you will need to add another if else block , so its solution is an A lookup table with Pointers. We'll have two arrays: one with the command names and one with the functions they point to.

#### what is forward Declarations ?
these lines you see at top , which looks like an function Declarations those are forward Declarations. We are about to create an array of functions (builtin_func) that uses these names.By declaring them at the top, we're telling the C compiler, "Trust me bro these functions exist. Trust me for now, and I'll define them later. 

then we have the **builtin_str[]** , its like an box which holds pointer to actual strings "cd" , "help" , "exit" , now to a bit confusing part the **Function Pointer Array** The syntax is bit confusing not bit its very confusing atleast it was for me , so that is basically an : An array of pointers, named builtin_func, where each pointer points to a function that takes a char ** as an argument and returns an int. i hope it clears things in simple manner 

| `builtin_str` (The "Keys") | `builtin_func` (The "Values") |
| :--- | :--- |
| `[0]` &rarr; `"cd"` | `[0]` &rarr; (Pointer to `jeet_cd` function) |
| `[1]` &rarr; `"help"` | `[1]` &rarr; (Pointer to `jeet_help` function) |
| `[2]` &rarr; `"exit"` | `[2]` &rarr; (Pointer to `jeet_exit` function) |


After all this i have just declared those built in commands ( cd , help , exit) that is fairly simple maybe look the cd one up for like those `chdir()` and `getenv()` .

## Last piece 
Okay, we've done all the hard parts.
* We have a `jeet_loop` that reads, parses, and executes.
* We have `jeet_read_line` to get user input.
* We have `jeet_split_line` to tokenize that input.
* We have `jeet_launch` to run external programs like `ls`.
* And we just built our `jeet_cd`, `jeet_help`, and `jeet_exit` builtins, along with the lookup tables (`builtin_str` and `builtin_func`).

But... how does our loop know *which* one to call? How does it decide between running `jeet_cd` (a builtin) and `jeet_launch` (for an external command)?

That's the job of the `jeet_execute()` function. It's the "traffic cop" of our shell. Our `jeet_loop` calls it, and `jeet_execute` is responsible for routing the command to the right place.

Thanks to all the setup we did, this function is actually super simple.

### The `jeet_execute()` Function

Here's the code. This is the final function that connects our builtin system to our process-launching system.

```c
int jeet_execute(char **args)
{
  int i;

  // checking if pressed enter
  if (args[0] == NULL) {
    return 1;
  }

  // If not empty, loop through our builtins
  for (i = 0; i < jeet_num_builtin(); i++) {
    // Compare the command to our list of builtin names
    if (strcmp(args[0], builtin_str[i]) == 0) {
      // If it's a match, run the BUILTIN function
      return (*builtin_func[i])(args);
    }
  }

  // If the loop finishes, it's NOT a builtin.
  // So, run it as an EXTERNAL program.
  return jeet_launch(args);
}

#### Lets break it down:

This functions logic is simple:

First, it checks if the command is empty (`args[0] == NULL`). If the user just hit Enter, it returns `1` to tell the main loop to continue.
If there is a command, it loops through our list of builtins ("cd", "help", "exit"). If it finds a match, it runs the corresponding function (like `jeet_cd`) and returns.
If the loop finishes and finds no match, it means the command isnt a builtin. In that case, it just calls `jeet_launch` to run it as an external program.

