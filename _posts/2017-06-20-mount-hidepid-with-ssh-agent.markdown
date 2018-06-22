---
layout: post
title: "Mount option hidepid with ssh-agent"
date: 2017-06-20 20:30
comments: true
---

## Problem Description

In my [dotfiles](https://github.com/tankywoo/dotfiles/blob/master/.common/other.sh#L27-L48), when I start a new bash login shell, I will check if a ssh-agent is already started, if not, start a new ssh-agent:

```bash
# Set up ssh-agent
SSH_ENV="$HOME/.ssh/environment"

function start_agent {
	echo "Initializing new SSH agent..."
	/usr/bin/ssh-agent | sed 's/^echo/#echo/' > "${SSH_ENV}"
	echo succeeded
	chmod 600 "${SSH_ENV}"
	. "${SSH_ENV}" > /dev/null
	/usr/bin/ssh-add;
}

# Source SSH settings, if applicable
if [ -f "${SSH_ENV}" ]; then
	. "${SSH_ENV}" > /dev/null
	#ps ${SSH_AGENT_PID} doesn't work under cywgin
	ps -ef | grep ${SSH_AGENT_PID} | grep ssh-agent$ > /dev/null || {
	start_agent;
}
else
	start_agent;
fi
```

When starting a new ssh-agent, I saved the info (environ variable) to a file, and next time I check the exists of the file, then check if ssh-agent process exists.

Recently, I login a jump server with one people one login user, and found a process owned by other people with command line:

```
redis-cli -h x.x.x.x -a <password string>
```

It's obvious that as a common permission user, I can also see other people's process with command line, if someone write password in the command line, not in interactive mode, it is insecure.


## Resolve the process list isolation

The solution is **mount with option `hidepid`**.

Such as `ps`, `top` etc. the process info is from `/proc`, in the head of `man 5 proc` document describe mount option `hidepid`, the default value is 0, and not specified when mount proc directory, all user can see other's processes command line.

With value 1, user can only see his own processes, but can list other's process id under `/proc` directory.

With value 2, user can only see his own processes, and if he list directory under `/proc`, he can only list his own pid directory, other pid directory is hidden.

So we can choose 1 or 2 to implement process list isolation:

```bash
# runtime
mount -o remount,rw,hidepid=2 /proc

# in /etc/fstab
# <fs>      <mountpoint>   <type>     <opts>               <dump/pass>
proc        /proc          proc       defaults,hidepid=2   0 0
```

Ok, do this can resolve the process list isolation, but it introduce another problem:

**I can't see the ssh-agent process**, this cause every time I start a new login shell, I will ask to start a new ssh-agent process with my dotfiles.

From user root:

```bash
root $ ps aux | grep ssh-agent
tankywoo 16282  0.0  0.0  19584   344 ?        Ss   18:03   0:00 /usr/bin/ssh-agent

root $ ls -al /proc/16282
ls -al /proc/16282
total 0
dr-xr-xr-x   9 tankywoo tankywoo 0 Jun 20 18:04 .
dr-xr-xr-x 214 root     root     0 Jun 14 02:52 ..
dr-xr-xr-x   2 tankywoo tankywoo 0 Jun 20 18:04 attr
-rw-r--r--   1 root     root     0 Jun 20 18:04 autogroup
-r--r--r--   1 root     root     0 Jun 20 18:04 cmdline
-r--r--r--   1 root     root     0 Jun 20 18:04 stat
-r--r--r--   1 root     root     0 Jun 20 18:04 statm
-r--r--r--   1 root     root     0 Jun 20 18:04 status
...
```

From the process's owner tankywoo:

```bash
tankywoo $ ls -al /proc/16282
ls: cannot open directory '/proc/16282': Operation not permitted
```

## Why are the files under /proc/[pid] owned by root?

As the result list above, files under /proc/16282 owns to root, not tankywoo.

We can `ps aux` and find other processes owns by yourself, and the files under `/proc/[pid]` should owns to yourself.

The I found in `man 5 proc`:

> `/proc/[pid]`
>
>   There is a numerical subdirectory for each running process; the subdirectory is named by the process ID.
>
>   Each `/proc/[pid]` subdirectory contains the pseudo-files and directories described below.  **These files are normally owned by the effective user and effective group ID of the process.  However
>   is made `root:root` if the process's "dumpable" attribute is set to a value other than 1.**  This attribute may change for the following reasons:
>
>   *  The attribute was explicitly set via the `prctl`(2) `PR_SET_DUMPABLE` operation.
>
>   *  The attribute was reset to the value in the file `/proc/sys/fs/suid_dumpable` (described below), for the reasons described in `prctl`(2).
>
>   Resetting the "dumpable" attribute to 1 reverts the ownership of the `/proc/[pid]/*` files to the process's real UID and real GID.

As above said, if the process's `dumpable` attribute is set with value not equal 1, the files will be owned by root.

More with `man 2 prctl`:

> PR_SET_DUMPABLE (since Linux 2.3.20)
> 
>   Set the state of the "dumpable" flag, which determines whether core dumps are produced for the calling process upon delivery of a signal whose  default  behavior  is  to
>   produce a core dump.
> 
>   **In  kernels  up  to and including 2.6.12, arg2 must be either 0 (`SUID_DUMP_DISABLE`, process is not dumpable) or 1 (`SUID_DUMP_USER`, process is dumpable).**  Between kernels
>   2.6.13 and 2.6.17, the value 2 was also permitted, which caused any binary which normally would not be dumped to be dumped readable by root only; for  security  reasons,
>   this feature has been removed.  (See also the description of `/proc/sys/fs/suid_dumpable` in proc(5).)
> 
>   Normally,  this  flag is set to 1.  However, it is reset to the current value contained in the file `/proc/sys/fs/suid_dumpable` (which by default has the value 0), in the
>   following circumstances:
> 
>   *  The process's effective user or group ID is changed.
> 
>   *  The process's filesystem user or group ID is changed (see credentials(7)).
> 
>   *  The process executes (execve(2)) a set-user-ID or set-group-ID program, or a program that has capabilities (see capabilities(7)).
> 
>   Processes that are not dumpable can not be attached via ptrace(2) PTRACE_ATTACH; see ptrace(2) for further details.
> 
>   If a process is not dumpable, the ownership of files in the process's /proc/[pid] directory is affected as described in proc(5).

Finally, I read the source code of `ssh-agent.c`, my openssh version is `openssh-7.5p1`.

Line 1421 of ssh-agent.c, for security, ssh-agent set ulimit's core dump to 0 to deny it.

```c
#ifdef HAVE_SETRLIMIT
  /* deny core dumps, since memory contains unencrypted private keys */
  rlim.rlim_cur = rlim.rlim_max = 0;
  if (setrlimit(RLIMIT_CORE, &rlim) < 0) {
      error("setrlimit RLIMIT_CORE: %s", strerror(errno));
      cleanup_exit(1);
  }
#endif
```

and line 1234, ssh-agent set `PR_SET_DUMPABLE`:

```c
// ssh-agent.c L1234
platform_disable_tracing(0);

// platform-tracing.c L33
void
platform_disable_tracing(int strict)
{
#if defined(HAVE_PRCTL) && defined(PR_SET_DUMPABLE)
    /* Disable ptrace on Linux without sgid bit */
    if (prctl(PR_SET_DUMPABLE, 0) != 0 && strict)
        fatal("unable to make the process undumpable");
#endif
...
}
```

Refer to the man document, `prctl(PR_SET_DUMPABLE, 0)` cause the files under `/proc/[pid]` owned by user root.


## Why I can't view `/proc/[pid]` of ssh-agent?

From `man 2 ptrace`, it list the algorithm of access to /proc/[pid]:

> 4.  Deny access if the target process "dumpable" attribute has a value other than 1 (`SUID_DUMP_USER`; see the discussion of `PR_SET_DUMPABLE` in prctl(2)), and the caller does not have the `CAP_SYS_PTRACE` capability in the user namespace of the target process.


## How to resolve the deny access of ssh-agent?

No solution I can think of at the moment...

In `sys/prctl.h` defined with `#define PR_SET_DUMPABLE   4`, unless we change the make of source code...

It is a sad story...
