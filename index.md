# Shell lab实验记录

在看视频的时候对进程和信号之间的原理有了更深的理解，但是还有一些不太懂的地方，希望通过这个实验去查漏补缺。  
实验需要补全未实现的代码，完成实验后就可以实现一个简易的shell程序，需要补全的函数如下所示
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

SIGINT： 通知前台进程终止进程(Ctrl+C)
SIGCONT: 继续执行一个停止的进程(kill)
SIGQUIT：与SIGINT类似，但是由QUIT字符控制(Ctrl-\)，进程在收到SIGQUIT信号时会产生core文件
SIGTSTP：停止进程的运行(Ctrl+Z)
SIGCHLD：父进程回收子进程的信号

int sigemptyset(sigset_t *set): 信号集初始化为空
int sigfillset(sigset_t *set)：把信号集初始化包含所有已定义的信号
int sigaddset(sigset_t *set, int signo)：把信号signo添加到信号集set中，成功时返回0，失败时返回-1
int sigdelset(sigset_t *set, int signo)：把信号signo从信号集set中删除，成功时返回0，失败时返回-1
int sigismember(sigset_t *set, int signo)：判断给定的信号signo是否是信号集中的一个成员，如果是返回1，如果不是，返回0，如果给定的信号无效，返回-1
int sigpromask(int how, const sigset_t *set, sigset_t *oset)：该函数可以根据参数指定的方法修改进程的信号屏蔽字，how的具体参数如下所示
    1、SIG_BLOCK       把参数set中的信号添加到信号屏蔽字中
    2、SIG_SETMASK     把信号屏蔽字设置为参数set中的信号
    3、SIG_UNBLOCK     从信号屏蔽字中删除参数set中的信号
```

## main
```
int main(int argc, char **argv) 
{
    char c;
    char cmdline[MAXLINE];//最长存储1024字节
    int emit_prompt = 1; //发出提示(默认)

    /* Redirect stderr to stdout (so that driver will get all output
     * on the pipe connected to stdout) */
    dup2(1, 2);//复制文件描述符

    /*分析命令行*/
    while ((c = getopt(argc, argv, "hvp")) != EOF) {
        switch (c) {
        case 'h':             /* 选择h则输出帮助内容 */
            usage();
	    break;
        case 'v':             /* emit additional diagnostic info */
            verbose = 1;
	    break;
        case 'p':             /* 不打印提示 */
            emit_prompt = 0;  /* handy for automatic testing */
	    break;
	default:
            usage();
	}
    }

    /* Install the signal handlers */

    /* shell对于ctrl-c, ctrl-z和SIGCHLD信号的接收以及反应 */
    Signal(SIGINT,  sigint_handler);   /* ctrl-c */
    Signal(SIGTSTP, sigtstp_handler);  /* ctrl-z */
    Signal(SIGCHLD, sigchld_handler);  /* Terminated or stopped child */

    /* 杀死shell进程 */
    Signal(SIGQUIT, sigquit_handler); 

    /* 初始化job列表 */
    initjobs(jobs);

    /* 执行shell的read/eval循环 */
    while (1) {

	/* Read command line */
	if (emit_prompt) {
	    printf("%s", prompt);
	    fflush(stdout);
	}
	if ((fgets(cmdline, MAXLINE, stdin) == NULL) && ferror(stdin))
	    app_error("fgets error");
	if (feof(stdin)) { /*对于shell而言命令行的结尾是ctrl-d*/
	    fflush(stdout);
	    exit(0);
	}

	/* Evaluate the command line */
	eval(cmdline);
	fflush(stdout);
	fflush(stdout);
    } 

    exit(0); /* control never reaches here */
}
```
在main函数中主要是对于命令行的解析、对ctrl-c/z的处理以及初始化job列表等操作

## eval
```
void eval(char *cmdline) 
{
    char *argv[MAXARGS];
    char buf[MAXLINE];
    int bg;
    pid_t pid;

    strcpy(buf, cmdline);
    bg = parseline(buf, argv);//解析命令行为一个数组，其中argv[0]是命令，argv[1]后面的是参数，当shell的命令以一个&结尾，则允许shell在后台运行此作业并无需等待其完成
    if (argb[0] == NULL)//得到空行
        return;

    if (!builtin_cmd(argv))//检测是否为内建命令，如果是则直接执行，若不是则意味着我们要求shell执行一些程序
    {
        if ((pid = Fork()) == 0)//子进程
        {
            if (execve(argv[0], argv, environ) < 0)//子进程调用execve函数，当execve有返回值的时候总是会返回-1(即出错)
            {
                printf("%s: Command not found.\n", argv[0]);
                exit(0);
            }
        }
        //当父进程获得使用权时候
        if (!bg)//如果不是一个后台进程
        {
            int status;
            if (waitpid(pid, &status, 0) < 0)//通过调用waitpid函数等待前台进程结束并回收
                unix_error("waitfg: waitpid error");
        }
        else//如果是一个后台进程则无需等待
            printf("%d %s", pid, cmdline);
    }
    return;
}
```
csapp公开课上给出的eval函数原型
