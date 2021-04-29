# Shell lab实验记录

在看视频的时候对进程和信号之间的原理有了更深的理解，但是还有一些不太懂的地方，希望通过这个实验去查漏补缺。  
实验需要补全未实现的代码，完成实验后就可以实现一个shell程序，需要补全的函数如下所示
```
eval: 对输入命令进行判断和执行
builtin_cmd: 检查是否为内置指令，是则直接执行
do_bgfg: 执行fg和bg指令
waitfg: 等待fg指令执行完毕
sigchld_handler: SIGCHLD信号处理函数
sigint_handler: SIGINT信号处理函数
sigtstp_handler: SIGTSTP信号处理函数
```

书上的一些笔记
```
getpid: 返回调用进程的PID
getppid: 返回调用进程的父进程的PID
pid_t在types.h中被定义为int
僵尸进程是父进程未回收的却已经终止的子进程，其仍然消耗内存资源。
init进程的pid为1，是所有进程的祖先，负责回收僵尸进程
一个进程可以调用waitpid函数来等待他的子进程终止或停止
fork调用一次返回两次，execve调用一次不返回

```

## eval
```
void eval(char *cmdline)
{
    char *argv[MAXARGS]; 
    char buf[MAXLINE];
    int bg;
    pid_t pid;

    strcpy(buf, cmdline);
    bg = parseline(buf, argv);
    if (argv[0] == NULL)
        return;

    if (!builtin_cmd(argv)) {
        if ((pid = Fork()) == 0) {
            if (execve(argv[0], argv, environ) < 0) {
                printf("%s: Command not found.\n", argv[0]);
                exit(0);
            }
        }

        if (!bg) {
            int status;
            if (waitpid(pid, &status, 0) < 0)
                unix_error("waitfg: waitpid erroe");
        }
        else
            printf("%d %s", pid, cmdline);
    }
    return;
}
```
上面这个是eval函数的原型，其会先对我们输入的命令进行判断是否为空和是否为内置函数，若是空则返回空，若不是内置函数则在子进程中调用execve函数。当execve返回-1（即报错）时，输出报错语句并退出。
若子进程中的命令顺利执行则继续判断前后台作业，前台作业则需要等待其执行完毕，后台作业则不需要等待。  
但是其本身并没有处理僵尸进程的功能，后续可以在其基础上对我们代码进行修改。 
## Fork
Fork函数可以在csapp.c中找到，在这里大写开头的函数都是对原函数的所作的error处理  
```
pid_t Fork(void) 
{
    pid_t pid;

    if ((pid = fork()) < 0)
    unix_error("Fork error");
    return pid;
}
```
## builtin_cmd
builtin_cmd命令中我们需要设置内置函数"quit", "jobs", "fg"和"bg", 特别的对于&命令则不做特别处理(return 1)  
```
int builtin_cmd(char **argv) 
{
    if (!strcmp(argv[0], "quit"))
        exit(0);
    if (!strcmp(argv[0], "jobs")) {
        listjobs(jobs);
        return 1;
    }
    if (!strcmp(argv[0], "bg")||!strcmp(argv[0], "fg")) {
        do_bgfg(argv);
        return 1;
    }
    if (!strcmp(argv[0], "&"))
        return 1;
    return 0;     /* not a builtin command */
}
```
## do_bgfg
do_bgfg函数实现了bg与fg的功能  
```

```
