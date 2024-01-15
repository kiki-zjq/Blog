# CMU 15213 Shell Lab

CMU 15213 的倒数第二个 Lab，对应着 CSAPP 这本书的信号与异常那一章。这个 Lab 的目的是为了让学生更加熟悉进程控制以及信号量。在这个 Lab 中，我们需要实现一个自己的命令行工具 tiny shel（类似于 Linux Shell），tiny shell 支持简单的 Job Control 和 IO 重定向。

在真正的开始之前，writeup 还提供了一些基本的知识

- Shell 中每个命令由一个或多个词组成，第一个词是要执行的动作名称。
    - 可以是 executable file path （例如，`tsh> /bin/ls`）
    - 也可以是一个内置的命令（例如，`tsh> quit`），后面几位是传递给该命令的命令行参数
- Shell 在子进程中运行每个 executable file
- 有两种类型的任务 —— foreground job 和 background job
- 如果说命令行以 `&` 结束，那么就是一个 background job
- 我们需要回收子进程，这里给出的建议是在 `sigchld_handler` 函数中完成这一步
- 我们的 Tiny Shell 需要可以捕获 `SIGINT` `SIGTSTP` 信号并将其发送给当前的 foreground job 的整个 process group
- 如果我们没有显式的对信号进行阻塞，那么 Signal Handlers 会非常任意的并发运行。这可能造成一些奇怪的 race conditions，因此我们需要在合适的地方完成信号的阻塞。

读完 writeup 后，我们的主要任务是完成以下函数

- `eval`: Main routine that parses, interprets, and executes the command line.
- `sigchld_handler`: Handles SIGCHLD signals.
- `sigint_handler`: Handles SIGINT signals (sent by Ctrl-C).
- `sigtstp_handler`: Handles SIGTSTP signals (sent by Ctrl-Z).

在了解了这些之后，就可以开始写代码了！这次 Lab 实际上思路都很简单直接，只需要按照测试用例一步一步实现即可

---

## Signal

这里先补充一点信号的基本知识

A ***signal*** is a small message that notifies a process that an event of some type has occurred in the system

- Similar to exceptions and interrupts
- Sent from the kernel to a process
- Signal type is identified by small integer ID’s (1 - 30)
- Only information in a signal is its ID and the fact that it arrives.

### Sending a Signal

***Kernel*** sends a signal to a ***destination process*** by *updating some state in the context of the destination process*. It will send a signal for one of the following reasons:

- Kernel has detected a system event such as divide-by-zero (SIGFPE) or the termination of a child process (SIGCHILD)
- Another process has invoked the ***kill*** system call to explicitly request the kernel to send a signal to the destination process

因此发送信号的流程可能是 `kernel → process`，也可以是 `process → kernel → process`

> ctrl + c 会发送一个 SIGINT 信号到前台进程组中的每一个进程，结果将是终止前台作业 —— 你输入 ps 看不到这个前台进程了
ctrl + z 会发送一个 SIGTSTP 信号到前台进程组中的每一个进程，默认情况是停止（挂起）前台作业 —— 你输入 ps 仍然可以看到这个前台进程
> 

### Receiving a Signal

A destination process ***receives*** a signal when it is forced by the kernel to react in some way to the delivery of the signal. Some possible way to react:

- ***Ignore*** the signal (do nothing)
- ***Terminate*** the process (with optional core dump)
- ***Catch*** the signal by executing a user-level function called signal handler

### Pending and Blocked Signal

- A signal is ***pending*** if sent but not yet received
    - There can be at most one pending signal of any particular type
    - Important, signals *are not queued*
        - If a process has a pending signal of type k, then subsequent signals of type k that are sent to the process are discarded

- A process can ***block*** the receipt of certain signals
    - Blocked signals can be delivered, but will not be received until the signal is unblocked

- A pending signal is received at most once

### Pending / Blocked Bits

Kernel maintains pending and blocked bit vectors in the context of each process

- ***pending bits***: represents the set of pending signals
    - Kernel sets bit k in pending when a signal of type k is delivered
    - Kernel clears bit k in pending when a signal of type k is received
    
- ***blocked bits***: represents the set of blocked signals

---

## Step 0. 实现阻塞信号的工具函数

为了在并发环境中提供信号处理的原子性和确定性。在我们的 Tiny Shell 解析完命令并且处理这些命令的时候，我们需要将相关的信号量给阻塞住。以此来

- **避免竞争条件:** 当多个进程或线程可能同时响应同一个信号时，如果不加控制地处理信号，可能会导致竞争条件。例如，如果在处理一个SIGCHLD信号的同时又收到了另一个SIGCHLD，可能会导致一些子进程的退出状态丢失。
- **保持状态一致性:** 在复杂的信号处理逻辑中，可能需要读写共享资源或状态。通过阻塞信号，我们可以确保在这些操作执行期间不会被信号中断，从而保持状态的一致性。

在这里我们可以先写几个工具函数。帮助我们快速创建一个新的 signal set，并且 block / unblock 信号量。

```c
/*****************
 * Signal Setters
 *****************/

/**
 * Setting up a new empty signal set.
 *
 * First we will initialize the signal set pointed to by mask to empty, with all signals excluded from the set.
 * And then we will add SIGCHLD, SIGINT, and SIGTSTP to the set.
 *
 * After these calls, mask would be a signal set that includes SIGCHLD, SIGINT, and SIGTSTP.
 * And we can use this signal set to block / unblock / or change the way these signals are handled by a process.
 *
 * @param mask
 */
void set_signals(sigset_t *mask) {
    sigemptyset(mask);
    sigaddset(mask, SIGCHLD);
    sigaddset(mask, SIGINT);
    sigaddset(mask, SIGTSTP);
}

/**
 * We will call sigprocmask function to block all signals that are currently in
 * the set pointed to by mask.
 * And this function will store the previous mask into prev_mask. So we can
 * restore (unblock) the signal set with the prev_mask.
 *
 * @param mask
 * @param prev_mask
 */
void block_signals(sigset_t *mask, sigset_t *prev_mask) {
    sigprocmask(SIG_BLOCK, mask, prev_mask);
}

/**
 * Unblock the signal set with the prev_mask.
 *
 * @param prev_mask
 */
void unblock_signals(sigset_t *prev_mask) {
    sigprocmask(SIG_SETMASK, prev_mask, NULL);
}
```

## Step 1. 实现一个基本的 `eval`

`eval` 函数中，我们需要解析输入的命令，根据 writeup，我们大致需要解决的命令类型包括 `Quit` `Foreground_Job` `Background_Job` `Builtin Jobs` `Path / Executable File`

start code 已经给出了完整的 `parse_result = parseline(cmdline, &token)` 的逻辑。我们可以通过 `token.builtin` 属性来确定当前命令的类型

```c
struct cmdline_tokens {
    int argc;               ///< Number of arguments passed
    char *argv[MAXARGS];    ///< The arguments list
    char *infile;           ///< The filename for input redirection, or NULL
    char *outfile;          ///< The filename for output redirection, or NULL
    builtin_state builtin;  ///< Indicates if argv[0] is a builtin command
    char _buf[MAXLINE_TSH]; ///< Internal backing buffer (do not use)
};

typedef enum builtin_state {
    BUILTIN_NONE = 8,  ///< Not a builtin command
    BUILTIN_QUIT = 9,  ///< `quit` (exit the shell)
    BUILTIN_JOBS = 10, ///< `jobs` (list running jobs)
    BUILTIN_BG = 11,   ///< `bg` (run job in background)
    BUILTIN_FG = 12    ///< `fg` (run job in foreground)
} builtin_state;
```

接下来就是一个简单的 switch case 逻辑（或者 if … else 也可以）

其中 FG Job 和 BG Job 其实有着相似的逻辑，因此这里我直接使用了一个 `handle_fgbg` 函数来处理，第一遍做这个 lab 的时候也可以先分开创建 handler 函数

```c
/**
 * @brief
 *
 * In this function, we will parse the commands input from the user. And based on
 * the type of command, call different handler functions.
 *
 * NOTE: The shell is supposed to be a long-running process, so this function
 *       (and its helpers) should avoid exiting on error.  This is not to say
 *       they shouldn't detect and print (or otherwise handle) errors!
 */
void eval(const char *cmdline) {
    parseline_return parse_result;
    struct cmdline_tokens token;
    int fd_in = STDIN_FILENO;
    int fd_out = STDOUT_FILENO;

    // Parse command line
    parse_result = parseline(cmdline, &token);

    sigset_t mask, prev_mask;

    /* When our command declared outfile */
    if (token.outfile != NULL) {
        fd_out = open(token.outfile, O_WRONLY | O_CREAT | O_TRUNC, DEF_MODE);
        if (fd_out < 0) {
            if (errno == EACCES)
                sio_printf("%s: Permission denied\n", token.outfile);
            else if (errno == ENOENT)
                sio_printf("%s: No such file or directory\n", token.outfile);
            return;
        }
    }

    /* When our command declared infile */
    if (token.infile != NULL) {
        fd_in = open(token.infile, O_RDONLY, DEF_MODE);
        if (fd_in < 0) {
            if (errno == EACCES)
                sio_printf("%s: Permission denied\n", token.infile);
            else if (errno == ENOENT)
                sio_printf("%s: No such file or directory\n", token.infile);
            return;
        }
    }

    if (parse_result == PARSELINE_ERROR || parse_result == PARSELINE_EMPTY) {
        return;
    }

    set_signals(&mask);

    /* Implement command handlers */
    switch (token.builtin) {
        case BUILTIN_QUIT:  // For Quit command, exit directly
            exit(0);
            break;
        case BUILTIN_FG:    // FG and BG command has similar logic
        case BUILTIN_BG:
            handle_fgbg(&token, mask, prev_mask);
            break;
        case BUILTIN_JOBS:  // Check the list of jobs
            block_signals(&mask, &prev_mask);
            list_jobs(fd_out);
            unblock_signals(&prev_mask);
            break;
        case BUILTIN_NONE:  // Consider as a path or executable file
            block_signals(&mask, &prev_mask);
            handle_none(fd_out, fd_in, token, prev_mask, cmdline, parse_result);
            unblock_signals(&prev_mask);
        default:
            break;
    }

    return;
}
```

## Step 2. `list_jobs`

当我们输入的指令是 `BUILTIN_JOBS` 时，我们会调用 `list_jobs` 函数，来展现当前的 jobs 列表。

由于 `list_jobs` 已经由 start code 给出，所以这项任务在上面的写法中已经完成了～

关于 job list，我们只需要后续注意调用 `add_job` 和 `delete_job` 函数将任务从 job list 中增删即可。

## Step 3. Signal Handler

观察 main 函数，里面有一部分的逻辑是注册 signal handler，它通过调用 `Signal` 函数，指明了当我们的程序收到指定的信号后，该使用什么函数来处理这些信号。

```c
Signal(SIGINT, sigint_handler);   // Handles Ctrl-C
Signal(SIGTSTP, sigtstp_handler); // Handles Ctrl-Z
Signal(SIGCHLD, sigchld_handler); // Handles terminated or stopped child
```

其中 `sigint_handler` 和 `sigtstp_handler` 都非常简单，可以在这里直接给出

```c
/**
 * @brief
 *
 * This function will handle SIGINT signal (Ctrl-C). The SIGINT signal will
 * interrupt and terminate a process.
 * We will check if there is a fg job exist, and then sends the SIGINT signal
 * to all processes in the foreground process group.
 *
 * The current value of errno is saved at the start and restored at the end to
 * ensure the handler does not interfere with the normal execution flow by changing errno
 *
 */
void sigint_handler(int sig) {  // Handle Ctrl-C -> DONE
    int _errno = errno;
    if (fg_pid > 0)
        killpg(fg_pid, sig);
    errno = _errno;
    return;
}

/**
 * @brief
 *
 * This function will handle SIGTSTP signal (Ctrl-Z). The SIGTSTP signal will
 * pause the current foreground job.
 * We will check if there is a fg job exist, and then sends the SIGTSTP signal
 * to all processes in the foreground process group
 *
 * The current value of errno is saved at the start and restored at the end to
 * ensure the handler does not interfere with the normal execution flow by changing errno
 *
 */
void sigtstp_handler(int sig) { // Handle Ctrl-Z -> DONE
    int _errno = errno;
    if (fg_pid > 0)
        killpg(fg_pid, sig);
    errno = _errno;
    return;
}
```

大致思路就是我们有一个全局变量 `fg_pid` 标识当前的前台任务，然后我们要判断当前是否存在一个前台任务，如果任务存在的话，我们就调用 `killpg` 函数将信号发送过去

## Step 4. `handle_none`

当我们输入的并不是一个 builtin command 的时候，我们需要考虑我们输入的可能是一个 path 或者 executable file，因此此时我们需要使用 fork 函数创建一个子进程来执行它。

此时，我们的思路其实是

1. child process 调用 `execv` 函数尝试执行当前的命令
2. parent process 判断这是一个前台任务 / 后台任务
    1. 前台任务：将任务加入到 job list，调用 `sigsuspend` 等待前台任务结束
    2. 后台任务：将任务加入到 job list，然后在屏幕显示相关信息即可

```c
/**
 * If not a builtin command, we will consider it as a path or executable file
 * Then we will create a new process try to run this file.
 *
 * If we define infile or outfile in our command, first we will run dup2 function
 * to redirection the pointer from the file descriptor to the file table.
 * And then we will try to execute the file.
 *
 * If it is a fg job, we will run it in the foreground explicitly.
 * And if it a bg job, we will run it in the backward and only print one line
 * information in the shell.
 *
 * @param fd_out
 * @param fd_in
 * @param token
 * @param prev_mask
 * @param cmdline
 * @param parse_result
 */
void handle_none(
    int fd_out,
    int fd_in,
    struct cmdline_tokens token,
    sigset_t prev_mask,
    const char *cmdline,
    parseline_return parse_result) {

    pid_t pid = fork();
    if (pid == 0) {  // In the child process

        sigprocmask(SIG_SETMASK, &prev_mask, NULL);  // unblock
        if (fd_in < 0 || fd_out < 0) {
            exit(1);
        }

        /* If we define outfile or infile, use dup2 function to complete the redirection logic */
        if (fd_out != STDOUT_FILENO) {
            dup2(fd_out, STDOUT_FILENO);
            close(fd_out);
        }

        if (fd_in != STDIN_FILENO) {
            dup2(fd_in, STDIN_FILENO);
            close(fd_in);
        }

        setpgid(pid, pid);

        /* Run the executable program */
        if (execv(token.argv[0], token.argv) < 0) {
            if (errno == EACCES)
                sio_printf("%s: Permission denied\n", token.argv[0]);
            else if (errno == ENOENT)
                sio_printf("%s: No such file or directory\n", token.argv[0]);
            exit(-1);
        }
    }
    else
    {   // In the parent process
        if (parse_result == PARSELINE_FG) {
            fg_pid = pid;       // Record current pid
            fg_is_running = FG_RUNNING;  // Mark fg job is running

            // add the fg job to the job list
            add_job(pid, FG, cmdline);

            // run the job & waiting for job completed
            while (fg_is_running == FG_RUNNING) {
                sigsuspend(&prev_mask);
            }
        }
        else if (parse_result == PARSELINE_BG)
        {
            add_job(pid, BG, cmdline);  // bg job -> add to job list only
            // print bg job information
            sio_printf("[%d] (%d) %s\n", job_from_pid(pid), pid, cmdline);
        }
    }

}
```

## Step 5. `sigchld_handler`

当 child process 在执行的时候，实际上就会触发我们的 `sigchld_handler` 

```c
/**
 * @brief
 * sigchld_handler function handle the `SIGCHLD` signal, which is sent to a process
 * when its child processes terminate or change state.
 *
 * In this function, we reap the child process by using waitpid function.
 * And based on different child states, we will print different info in the shell.
 *
 */
void sigchld_handler(int sig) {
    int _errno = errno;
    sigset_t mask, prev_mask;

    /* Block signals */
    set_signals(&mask);
    block_signals(&mask, &prev_mask);

    int statusp;
    pid_t pid;
    while ((pid = waitpid(-1, &statusp, WNOHANG | WUNTRACED | WCONTINUED)) > 0) {
        jid_t jid = job_from_pid(pid);
        if (WIFCONTINUED(statusp)) {
            /*
             * If we receive `fg` or `bg` command, we will come into this branch
             * `fg` or `bg` will turn a stop job to a FG job or BG job.
             * So first we get the job's current state `jstate`
             * If `jstate` is ST, then based on the command we receive, we will
             * turn this job to a FG job / BG job.
             */
            job_state jstate = job_get_state(jid);
            if (jstate == ST) {
                if (st_to_fgbg == TO_FG) {
                    job_set_state(jid, FG);
                } else if (st_to_fgbg == TO_BG) {
                    job_set_state(jid, BG);
                }
            }
        }
        else
        {
            if (pid == fg_pid) {
                // foreground job ended, set fg_is_running to zero
                fg_is_running = FG_NO_RUNNING;
            }

            if (WIFSIGNALED(statusp)) { // SIGINT
                int signum = WTERMSIG(statusp);
                delete_job(jid);
                sio_printf("Job [%d] (%d) terminated by signal %d\n", jid, pid, signum);
            }

            if (WIFSTOPPED(statusp)) { // SIGTSTP
                int signum = WSTOPSIG(statusp);
                job_set_state(jid, ST);
                sio_printf("Job [%d] (%d) stopped by signal %d\n", jid, pid, signum);
            }

            if (WIFEXITED(statusp)) { // exit() or return from main()
                delete_job(jid);
            }
        }
    }

    /* Unblock signals */
    unblock_signals(&prev_mask);

    /* Reset the error number */
    errno = _errno;
}
```

## Step 6. `handle_fgbg`

```c
/**
 * Handle FG / BG command.
 * 1) First we will try to get the job id or pid from the command.
 *    And then we will get the job id based on the pid / or get the pid based on job id.
 * 2) Second, based on the command type, we will run similar but different logic:
 *      2.1) If it is a `bg` command. Send SIGCONT signal directly.
 *      2.2) If it is a 'fg' command.
 *          - If target job is ST state, then send a SIGCONT signal
 *          - If target job is a background job, set state to FG
 *
 * @param token
 * @param mask
 * @param prev_mask
 */
void handle_fgbg(struct cmdline_tokens *token, sigset_t mask, sigset_t prev_mask) {

    /* check if the command is valid */
    if (token->argc != 2) {
        if (token->builtin == BUILTIN_FG)
            sio_printf("fg command requires PID or %%jobid argument\n");
        else
            sio_printf("bg command requires PID or %%jobid argument\n");
        return;
    }

    /* get value from the command */
    const char *type = token->argv[0];  // `bg` or `fg`
    const char *id = token->argv[1];    // pid or jid

    /* block the signal */
    block_signals(&mask, &prev_mask);

    /*
     * Get jid and pid for this job
     * 1). If the id start with '%', the command gives us job id. We should get
     *     pid based on this job id.
     * 2). Else if the id start with a number, the command gives us a pid.
     *     We should get job id based on this pid.
     * 3). Print an error message.
     */
    pid_t pid;
    jid_t jid;
    if ('0' <= id[0] && id[0] <= '9')
    {
        pid = (pid_t)atoi(id);
        jid = job_from_pid(pid);
    }
    else if (id[0] == '%')
    {
        jid = (jid_t)atoi(&id[1]);
        if (job_exists(jid) == false) {
            sio_printf("%s: No such job\n", id);
            unblock_signals(&prev_mask);
            return;
        }
        pid = job_get_pid(jid);
    }
    else
    {
        sio_printf("%s: argument must be a PID or %%jobid\n", type);
        unblock_signals(&prev_mask);
        return;
    }

    /* Get target job state, and move target job to fg / bg */
    job_state curState = job_get_state(jid);
    if (token->builtin == BUILTIN_BG)
    {
        st_to_fgbg = TO_BG;
        sio_printf("[%d] (%d) %s\n", jid, pid, job_get_cmdline(jid));
        killpg(pid, SIGCONT);
    }
    else if (token->builtin == BUILTIN_FG)
    {
        fg_pid = pid;
        fg_is_running = FG_RUNNING;

        if (curState == ST)
        {
            st_to_fgbg = TO_FG;
            killpg(pid, SIGCONT);
        }
        else if (curState == BG)
        {
            job_set_state(jid, FG);
        }

        /* Wait until the current FG job stop running */
        while (fg_is_running == FG_RUNNING) {
            sigsuspend(&prev_mask);
        }
    }

    /* unblock the signal set */
    unblock_signals(&prev_mask);
}
```