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
## eval
在视频的ppt中有给出eval代码的原型
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

    if (!builtin_command(argv)) {
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

