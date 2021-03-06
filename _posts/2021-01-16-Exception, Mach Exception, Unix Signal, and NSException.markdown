---
layout: post
title:  "Exception, Mach Exception, Unix Signal, and NSException"
date:   2021-01-16 21:10:10 +0800
categories: jekyll update
---

## Exception
Exception是一种用来实现特殊控制流的机制。特殊控制流用来跳出当前控制流，进入内核代码，执行相应的内核逻辑。

- Interrupt
	- 
- Trap
	- 陷入包括很多种。其中一种用来实现`系统调用`，假设它的编号是n。系统调用的时候参数被放入寄存器，然后通过汇编指令`int n`调用陷入n，来使用系统调用。例如，socket函数是对系统调用的一个函数封装，可以说是“系统级函数”在未陷入系统内核之前，刚开始调用这个函数的时候，你还是处在用户态，然后CPU保存现场，进行上下文切换，通过查找Exception Table进入对应的内核函数。
- Faults
	- 例如页面错误，产生页面错误时，返回地址为当前“过错指令”地址。
- Abort

## Mach Exception
> [https://www.mikeash.com/pyblog/friday-qa-2013-01-11-mach-exception-handlers.html](https://www.mikeash.com/pyblog/friday-qa-2013-01-11-mach-exception-handlers.html)


Mach Exception是Mach内核发出的一种消息，告诉相应的`Mach Task (Process)`有内核事件发生，需要处理。处理器在处理某些错误时，会通过特殊控制流机制调用内核代码，然后内核生成对应的Mach Exception并通过Mach IPC机制报告给对应的进程。

Mach Exception还被用来实现out-of-process debugging，例如LLDB(Low Level Debugger)可以通过`ptrace系统调用(使用PT_ATTACHEXC)`来attach到另一个进程，并通过`mach_vm_write`来修改`__TEXT`区，将目标地址的内容设置为`INT 3 (Trap)`来实现断点功能，取消断点时，也需要将内容恢复，因此更改前需要先记住内存地址的内容。

```md
实际上，还有许多其他步骤：
> [https://bugs.openjdk.java.net/browse/JDK-8189429]

The high level steps for attaching to a target process are somewhat as follows: 

* Allocate an exception port for receiving exception messages on behalf of the target process. 
* Add the ability to send messages on this exception port. 
* Save the list of exception ports of the target process (so that we restore it later while detaching from the process). 
* Register the newly allocated exception port with the target process. 
* Invoke ptrace with PT_ATTACHEXC on the target process. 
* Wait for the exception message from the kernel with the mach_msg() call by listening in on the newly created exception port. 
* Once the reply is obtained, call mach_exc_server() to parse and decode the kernel exception message, and to prepare a reply which would need to be sent by SA to the target process while detaching. Else the target would continue to remain suspended in the kernel. 
* mach_exc_server() resides in code generated by running the mig(1) command. The generation of this code is included in the build. mach_exc_server() invokes the catch_mach_exception_raise() routine which has to be implemented in SA. 
* Implement the catch_mach_exception_raise() routine to check that the message denotes a Unix soft signal, and return a "KERN_SUCCESS" (which would cause the generated mach_exc_server() routine to create a reply message indicating that this exception is being handled). 
* Suspend all the threads in the target task with task_suspend(). 

The steps to be followed while detaching: 

* Restore the pre-saved original exception ports registered with the target process. 
* Invoke ptrace() with the "PT_DETACH" request on the target process. 
* Reply to the previous "mach soft signal" exception, since unless this acknowledgement is sent, the thread raising the exception (in the target) remains suspended. The reply message to be sent is obtained from the previous call to mach_exc_server(). 
* Release the exception port allocated while attaching to the target using mach_port_deallocate(). 
```

## Unix Signal

## NSException
这是Objective-C语言层面异常，当发生这种异常时，如果没有捕获，语言会自动调用`abort系统调用`，此时进程会收到来自系统的SIGABRT信号。如果被捕获了，则需要进行`stack unwinding`。
> [https://www.mikeash.com/pyblog/friday-qa-2017-08-25-swift-error-handling-implementation.html](https://www.mikeash.com/pyblog/friday-qa-2017-08-25-swift-error-handling-implementation.html)

*语言设计者认为在Swift中`throw Error`会是一个合理的常见场景，因此在Swift中`throw Error`不需要进行栈展开。与之相应的是Swift在平时就付出了一定的运行时开销来记录和检查每个可throw调用的结果。*

如果你使用在Swift中使用`fatalError`，你会发现你最终收到了一个SIGABRT。