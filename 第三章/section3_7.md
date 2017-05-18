## **3．7系统调用及应用**

&emsp;&emsp;以下是用户态下模拟执行命令的一个示例程序。父进程打印控制菜单，并且接受命令，然后创建子进程，让子进程去处理任务，而父进程继续打印菜单并接受命令。

    #include <stdio.h>                                                                                                                                           
    #include <stdlib.h>
    #include <signal.h>     
    #include <sys/types.h>  
    #include <sys/wait.h>   
    #include <string.h>

    int main(int argc, char *argv[])
    {
        pid_t pid;
        char cmd;
        char *arg_psa[] = {"ps", "-a", NULL};
        char *arg_psx[] = {"ps", "-x", NULL};

        while (1) {
                printf("------------------------------------------------\n");
                printf("输入 a 执行'ps -a'命令\n");
                printf("输入 x 执行'ps -x'命令\n");
                printf("输入 q 退出\n");
                cmd = getchar();     /*接收输入命令字符*/
                getchar();

                if ((pid = fork()) < 0){//创建子进程
                        perror("fork error:");
                        return -1;
                }//进程创建成功
                if(pid == 0) {      /*子进程*/
                        switch (cmd) {
                                case 'a':
                                        execve("/bin/ps", arg_psa, NULL);
                                        break;
                                case 'x':
                                        execve("/bin/ps", arg_psx, NULL);
                                        break;
                                case 'q':
                                        break;
                                default:
                                        perror("wrong cmd:\n");
                                        break;
                        }  /*子进程到此结束*/
                        exit(0);  /*此处有意设置子进程提前结束，因为它的任务已经完成
                } else if (pid > 0) {  /*父进程*/
                        if ( cmd == 'q' )
                                break;
                }
        }  /*进进程退出循环*/
        while(waitpid(-1,NULL,WNOHANG) > 0);/*父进程等待回收子进程*/
        return 0;
    }

