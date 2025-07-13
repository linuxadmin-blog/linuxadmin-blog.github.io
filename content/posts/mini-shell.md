---
title: "Building a mini UNIX Shell"
date: 2025-07-12
author: Burak Kirik
tags: ["System Programming", "UNIX", "Shell"]
categories: ["System Programming"]
draft: false
---

One of the best ways to to learn how things work is to try and build them from scratch. That's exactly why I decided that I should build a mini shell so I can grasp the foundational concept of how a shell works. I personally like to read "development journals" with some extra storytelling spice in them, so why not try and make one myself?

Of course, to succeed in this project I actually needed to understand what exactly is a shell?
A shell is a yet so simple concept in it's core that when you are using it you might not even care about what goes on under the hood, it just works.

# What is a Shell?

A shell is basically a system program whose job is to provide a communication point to the Operating System for the user. It is a command interpreter, it takes user input and somehow translate that to processes to be carried out by the operating system.

It's as simple as that: A shell creates processes according to the user input.

Of course, advanced shells like `PowerShell`, `Bash`, `Zsh` provide many more functionalities but in the end, the core concept of a shell is that it provides you access to communicate with the operating system.

Unfortunately, the terms "Shell" and "Terminal" or "Terminal Emulator" are sometimes used interchangeably by mistake. Terminal is actually historical term in computing, it was a hardware device used to interact with a computer. It typically consisted of a keyboard for input and a display or a printer for output. Although we ditched terminals a long time ago the terminology sticked with us so far.

"Terminal Emultor" does exactly what it suggests, it provides with you a very simple form of interaction with a shell by emulating a terminal. Terminal Emulator is a software that mimics the behavior of the physical terminals and lets you interact with shells.

Typically a shell follows internally what is called a `RPEL` structure which stands for **Read**, **Parse**, **Execute**, **Loop**. This is the core principle of any interactive shell, it reads lines from the standard input (`stdin`), it parses them into meaningful tokens and it executes the command with specified arguements. Then it performs this loop until user decides to break it.

# Reading Input

Reading Input is probably the easiest part of a shell to implement, input is given by writing a line to stdin which a user can easily perform by typing in the input string and hitting `enter` in his terminal emulator.

When I looked up the best way to read a line from `stdin` with C, I stumbled upon this amazing function called `getline`. It's a relatively new C library function implemented in 2010. This function actually solves a really annoying problem in C, dynamically allocating the buffer needed to store the read input from the file stream. **Because we don't actually know how many characters of input we are going to read** (it highly depends on what the user enters) we can't really allocate the needed buffer with the exact size beforehand.

```C
char buf[1000];
fgets(buf, 1000, stdin);
```

this is really problematic, we create a static sized buffer (1000 characters) in the stack and try to read a line from stdin into the buffer.

**What if it's size is insufficent?** <br>
**What if the user needs to enter more characters?**

or even worse (and more likely), **What if the user enters only a few characters?** than most of the buffer is not used and wasted.

what **getline** internally does is, it creates the buffer in the **heap**, and it reads a line from a file stream *character by character*, as the given buffer becomes insufficent for storing new characters it then reallocates it to a bigger one.

here is how I implemented `sh_read_line` using `getline`

```C
char *sh_read_line(FILE *stream){
    char* line = NULL;
    size_t buf_size = 0;

    /*
        get_line will dynamically allocate and resize the buffer for line
        returns -1 on failure or when encountered EOF (might be caused by a signal like CTRL+D)
        returns the no. of characters read on success.
    */
    int res = getline(&line, &buf_size, stream);

    if(res == -1){
        free(line);
        return NULL;
    }
    else{
        return line; // Free line after done using it to prevent memory leaks!
    }
}
```

Seems fairly simple, though there are a couple of things to compensate:

`getline` expects us 3 things, a buffer (pointing to NULL) to store the read line, a `size_t` to store the no. of read characters and a file stream to read input from. I could actually directly pass `stdin` as the third parameter since this is what a shell should do. But I decided that I should keep it more flexible by taking in the file stream as a parameter. This might actually come in handy when you are working with non-terminal shells like `.sh` scripts.

Other than that, you probably noticed that `getline` returns an integer based on success or failure. When it returns `-1` it means that it failed to read a line into our buffer from the specified file stream. A really important point here is: when we check if `getline` failed and it actually did; before returning from the function we need to free the created buffer. Because `getline` creates this buffer in the `heap` meaning that it will actually stay there unused even after the function returns which will cause a memory leak.

Another important point is when the user presses `CTRL-D` in the terminal, he/she will actually write an EOF (End of File) to the `stdin`, which will cause `getline` to fail, this case needs to be handled specifically; though we will implement it in another place.

# Tokenizing Input

A line of string on it's own doesn't really mean much from the perspective of a shell, The shell needs to give meaning to this line. Converting the raw string to a form that the shell can "understand" is called `parsing`. But for convenience, before parsing the string we need to separate it into arguements as the user intended to, called `tokenizing`. This step may have been implemented as an internal part of parsing, but I don't really like functions or parts of code to get overcomplex. 

I've yet to see any shell that doesn't use `' '` (space) as a delimeter to separate arguements in a command, So that's what we are going to use as a delimeter.

An example of how this tokenization would work is as follows:

`input: "ls -l -a -h ~/shell_project"`</br> </br> 
`output: ["ls", "-l", "-a", "-h", "~/shell_project"]`

The input string is separated into individual sub-strings using the delimeter and pushed into an array.
The reason we use an array to store the tokens will become more obvious in the "Execution" chapter, mainly because of how the `exec` system call works.

Now in order to do this, the first thing that might come to your mind is the C library function `strtok` that exactly does this. `strtok` tokenizes a string with respect to a given delimeter.

Well, if you thought of this, you wouldn't really be wrong. This function might work just well for a really simple shell.

But a shell command isn't usually just any string, it's ,well, a shell command.

A shell command has special features that needs to be taken into account while tokenizing, such as:

1. **Single and Double quotes:** Commands might include strings or arguements that have spaces in them. Arguements inside quotes should be taken as single arguements rather than getting tokenized individually.

    Think of the command `echo "Hello World"`, logically this command should be tokenized into an array as:
    `["echo", "\"Hello World\""]`, Because "Hello" and "World" are not separate arguements in this command but they are one, a string that has a space character in it. `strtok` would consider this space as a valid delimeter and tokenize it as: `["echo", "\"Hello", "World\""]`.

2. **Paranthesis:** Shell commands may have sub-commands `$(command)` in them, They should also be taken as single tokens to be recursively dealt with.

    An example might be:

    `echo $(ls -1)`, in the execution step, `echo` and `ls` should be different processes, `echo` should wait for the termination of the `ls` process and prompt it's result.

    Logically, with respect to this process structure, sub-commands should be taken as single tokens for convenience.

3. **Special characters and escapes:** special characters like `\n`, `\t`, `\` (escape space) should be dealt with specifically which strtok is incapable of.

Having all these reasons and more force us to write our of tokenization function. Though keep in mind that in this walkthrough we are just making a really simple shell so at this stage, none of the features provided above are really built into it. Though in order to keep an open door for future development it's still more sensible to write our own tokenization function.

Let's take a look at `sh_tokenize_line`

```C
Dynamic_Array *sh_tokenize_line(const char *line)
{
    const char *p = line;
    const char* start = line; // Set the start pointer (To be used inside the loop)
    Dynamic_Array *tokens = init_dynamic_array(DEFAULT_TOKEN_COUNT);

    if(tokens == NULL){
        return NULL;    // Terminate if failed to initialize tokens array
    }

    do
    {
        // When current character is whitespace, skip the character 
        while (isspace(*p)){p+=sizeof(char);} // Skip extra white spaces
        if(*p == '\0'){
            break;  // Skip the rest of the loop if '\0' is after white spaces
            /*
            Example on what this condition avoids: 
            "--> Hello World    \0"
            Without this condition the tokens would be: ["Hello\0"], ["World\0"], ["\0"]
            */
        }

        if(p != line && isspace(*(p - sizeof(char))) ){ // Encountered a regular character after white space character (start of a token), and we are not in the first iteration
            start = p;
        }
        while(!isspace(*p) && *p != '\0' ){p+=sizeof(char);} // Skip all regular characters, skip to the next white space

        size_t size_ch = p - start;
        char token[size_ch + 1]; // p - start = no. of read characters, including the last white space, which should be replaced by '\0'
        
        strncpy(token, start, size_ch);
        token[size_ch] = '\0';
        
        dynamic_array_push(tokens, token); // Put the obtained token in the dynamic array, + 1 accounts for null terminator because it's not counted by strlen
        
        if(*p != '\0'){
            p+=sizeof(char);
        }
    } while (*p != '\0');

    dynamic_array_equalize(tokens, DYNAMIC_ARRAY_FINALIZE);
    return tokens;
}
```

This might look a little bit daunting, in fact, after looking at this code after some time writing it, I had a bit hard while.

Let me explain this code. So we have a single parameter in this function, which is a `const char*` this is expected to be the line of string read with the `sh_read_line` function we implemented previously.

we have 2 character pointers `p` and `start`. The intention here is `p` should traverse the input string character by character. When we encounter a white space we are either at the start or the end of a token. `start` should point to start of tokens accordingly.

And also we use a dynamic array to store the tokens. That's just an array that uses dynamic memory. Since we don't know the size of the string and the number of tokens the user is going to pass in, we need a dynamic array that grows in size if it's full as new elements are pushed 

The dynamic array reallocates it's buffer to a heap double the size when it's full and a new element is pushed into it. 

This leaves some unused memory in most cases, so the `dynamic_array_equalize` function again reallocates to a new heap that matches the exact size of the elements it stores. This action should be done when we are **sure** we have all the elements we need in the array.

Back to the main subject: we have a while loop that traverses the given string until encountering a null-terminator character (`\0`) which marks the end of the string.

`while (isspace(*p)){p+=sizeof(char);}` moves `p` to point to the next character as long as `p` is a white space, meaning we skip all white spaces until we encounter a regular character. This also useful to remove leading or repeating white spaces and have a more flexible syntax.

"&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`ls .`" is a valid syntax as "`ls .`" is.

going on we have the statement:
```c
if(*p == '\0'){
            break;  // Skip the rest of the loop if '\0' is after white spaces
            /*
            Example on what this condition avoids: 
            "--> Hello World    \0"
            Without this condition the tokens would be: ["Hello\0"], ["World\0"], ["\0"]
            */
        }
```

This condition avoids takin the null-terminator as a token if it comes after a space (this happens when there are trailing spaces in the command), which again provides some more syntax flexibility.

consider the string:

"`ls .`&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`\0`"

when `p` encounters a null-terminator after spaces it will not evaluate it at all and break the loop since it marks the end of the string. If we dont have this condition, we would see `\0` as a token in our tokens array, which would cause issues later on when we try to use these tokens in the `exec` system call.

`exec` system call specifically needs an array of strings each terminated with `\0` and the array itself terminated with `NULL`.

Going on with analyzing the code:
```c
 if(p != line && isspace(*(p - sizeof(char))) ){ // Encountered a regular character after white space character (start of a token), and we are not in the first iteration
    start = p;
}
while(!isspace(*p) && *p != '\0' ){p+=sizeof(char);} // Skip all regular characters, skip to the next white space
```
the provided statement works when p encounters a regular character (white spaces were skipped and the previous character is a white space). It marks the start of a token with the `start` pointer. Than it increments `p` until a white space character is reached, which marks the end of a token.

so we have a token now, `start` points to the start of the token and currently `p` points to the end of the token.

`p - start` is a little pointer arithmetic that gives us the size of the token we obtained.

and after ending this token with a `\0` null terminator, we are ready to push it into the tokens array.

I know this might be a little complex, that's why I've made an animation for my visual learners out there:

![Animation](/static/img/mini-shell/H.gif) <!--TODO: Arrange the path before pr-->

# Parsing the Tokens

Tokenizing was the more complex part, and now we are left with the easier step to prepare our input command for execution: Parsing.

`Parsing` in this program's context means getting the obtained tokens in a special "Command" data structure we made.

The particular data structure is:

```c
typedef struct sh_command{
    char* name;  // name of the command e.g. "ls" or "/bin/ls", basically the first token in the tokens array
    char** argv; // argv, tokens themselves
    int argc;
}SH_Command;
```

as you can see a command is structured as: `name` (the name of the command which is the very first token in the array), `argv` the entirety of the tokens array and `argc`, number of items in the tokens array

the code that parses an array of tokens into this data structure is quite simple.

```c
SH_Command *sh_parse_tokens(Dynamic_Array tokens){
    SH_Command *command = (SH_Command*) malloc(sizeof(SH_Command));
    if(command == NULL){
        return NULL;
    }

    command->argv = tokens.arr;
    command->name = tokens.arr[0];
    command->argc = tokens.len - 1; // Assuming the array is a finalized one, the last element of the array would be NULL

    return command;
}
```

you might get confused with the line, `command->argc = tokens.len - 1;` that is because the `tokens` array is finalized with a `NULL` pointer. That is how the `exec` system call that we are going to use awaits it to be. Although it should still stay in the `argv` array, we should exclude it while accounting for the number of elements in the array as it's just a placeholder/end-marker.

# Executing the Command

Okay, so we've got our full command armed and ready. All good, but now; how do we really execute it. Well, being a simple user from the CPU's perspective we can't really directly make it create processes for us since it's a high privileged action. We need to ask the operating system to do that for us. That's where system calls take place. This is the main concept that makes our shell platform specific, because we are dependant on UNIX system calls.

2 of them we need to create a process with the process image we desire.

1. `fork()`
2. `exec()`

`fork()` will create a child process with the same text section as the parents, in fact the child process will inherit most features of the parent process.

of course the process that creates child processes here is our shell process.

`exec` system call is actually not a single type of system call, it's actually a family of system calls including various types such as:

- `execl()`
- `execv()`
- `execle()`
- `execlp()`
- `execvp()`

you can look into each and what they are good for, but all of them serve the same purpose: switch current process' process image with the desired one.

particularly, we are going to be using `execvp()` because it is the most robust one.

`int execvp(const char *file, char *const argv[]);`

it will take 2 parameters, path to an executable file. This might be the full path such as `/usr/bin/ls` or just a name, we can pass in just `ls` and it will search the `PATH` to find the location of the program.

the second one is `argv`, a `NULL` terminated array containing all of our arguements.

So here is how we utilize these 2 system calls to create an interactive shell environment:

```c
int sh_execute_command(SH_Command command)
{
    pid_t pid;
    pid = fork(); // Create a child process that is the copy of the current process

    if(pid < 0){ // if pid < 0 is returned, fork failed and we couldn't create a child process
        fprintf(stderr, "Couldn't create a child process for %s\n", command.name);
        return ERROR_GENERAL;
    }
    // Both child and parent processes continue execution from here
    else if (pid == 0){  // pid = 0 returned for the child process
        execvp(command.name, command.argv); // Replace the child's process image with the context of the command
        
        // If execvp fails, the process image is not replaced, thus it will continue executing instructions from here
        // The OS will reclaim everything that belongs to the child process after exiting
        exit(EXEC_FAIL);
    }
    else{ // pid > 0 (pid of the child process), which means that we are in the parent (current shell) process
        int status;
        waitpid(pid, &status, 0); // Wait for the given pid to exit, and write it's exit status code to status

        if(WIFEXITED(status) && WEXITSTATUS(status) == EXEC_FAIL){ // if the child process exited with EXEC_FAIL status code return failure
            fprintf(stderr, "Couldn't replace process image for %s\n", command.name);
        }
        return SUCCESS;
    }
}
```

if you are not sure how this code works, please take a look at the documentation for `fork`.

Though, another good thing to mention here is the `wait()` system call.

`waitpid(pid, &status, 0)` will wait for the process with the provided `pid` to terminate.

So in the parent shell process, we explicitly put this so the shell process doesnt continue on reading and parsing new input. It will wait for the child process to terminate it's execution to continue on.

# Putting it all together

We have all our necessary functions to create a shell. Now we can put them all together in `main()`

```c
int main(){
    /*
        Determine if the shell should in interactive mode.
        isatty(STDIN_FILENO) returns 1 if the provided file descriptor (STDIN_FILENO = 0) is a terminal

        If we are in interactive mode display a prompt
    */
    if(isatty(STDIN_FILENO)){ 

        while (1)
        {
            printf("$$ "); // Prompt

            char* line = sh_read_line(stdin);
            if(line == NULL){
                if(feof(stdin)){
                    break;
                }
                fprintf(stderr, "Shell failed to read a line from stdin\n");
            }
            else{
                Dynamic_Array *tokens = sh_tokenize_line(line);
                free(line);

                SH_Command *command = sh_parse_tokens(*tokens);

                sh_execute_command(*command); 

                free_dynamic_array(tokens);  // now command->name and command->argv are dangling pointers
                command->name = NULL;
                command->argv = NULL;
                free(command);
            }
        }
        return 0;
    }
    else{
        printf("Non-Interactive Mode\n");
    }
    return 0;
}
```

A key thing to mention here is how we determine if we should invoke an interactive shell or a non-interactive one (that will execute the given command once and terminate).

Well, if we are in a terminal we would most likely want to use an interactive shell.

We determine if we are in a terminal environment with `isatty` this library function will look to our `stdin` file descriptor to determine if it's a terminal.

Than, it will work in an endless loop to read, tokenize, parse and execute comands. That is of course until we send a `SIGINT` signal with `^C` or an `EOF` with `^D`

keep in mind that we should always free up heap memory when we are done using it. I've checked this entire program with `valgrind` and there were no memory leaks.

As of now, I didn't implement Non-Interactive Mode and actually this is a really small project but by design, it's left for further development and improvement. You can find the source code at this [repo](https://github.com/kirki58/shell). Please feel free to fork me!