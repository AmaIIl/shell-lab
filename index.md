# Shell lab实验记录

在看视频的时候对进程和信号之间的原理有了更深的理解，但是还有一些不太懂的地方，希望通过这个实验去查漏补缺。  
实验需要补全未实现的代码，完成实验后就可以实现一个shell程序，需要补全的函数如下所示
```
eval: 主要功能是解析cmdline，并且运行. [70 lines]
builtin cmd: 辨识和解析出bulidin命令: quit, fg, bg, and jobs. [25lines]
do bgfg: 实现bg和fg命令. [50 lines] 
waitfg: 实现等待前台程序运行结束. [20 lines]
sigchld handler: 响应SIGCHLD. 80 lines]
sigint handler: 响应 SIGINT (ctrl-c) 信号. [15 lines] 
sigtstp handler: 响应 SIGTSTP (ctrl-z) 信号. [15 lines]
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
