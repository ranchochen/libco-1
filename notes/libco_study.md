### libco分析 ###
此文主要七拼八凑介绍libco协程的关键原理，libco的优势在于其协程的优点。首先介绍下协程。

1. 协程
 
	1.1 什么是协程

	> “子程序就是协程的一种特例。” -- Donald Knuth

	协程，又称微线程，纤程。英文名Coroutine。

	协程的概念很早就提出来了，但直到最近几年才在某些语言（如Lua）中得到广泛应用。

	子程序，或者称为函数，在所有语言中都是层级调用，比如A调用B，B在执行过程中又调用了C，C执行完毕返回，B执行完毕返回，最后是A执行完毕。

	所以子程序调用是通过栈实现的，一个线程就是执行一个子程序。

	子程序调用总是一个入口，一次返回，调用顺序是明确的。而协程的调用和子程序不同。

	<b>协程看上去也是子程序，但执行过程中，在子程序内部可中断，然后转而执行别的子程序，在适当的时候再返回来接着执行。</b>

	注意，在一个子程序中中断，去执行其他子程序，不是函数调用，有点类似CPU的中断。

	1.2 协程的优势

	+ 因为子程序切换不是线程切换，而是由程序自身控制，线程是操作系统执行的最小单位，对于操作系统来说，他只能看到线程，不知道这个线程里面是怎么调度的。 因此，没有线程切换的开销，和多线程比，线程数量越多，协程的性能优势就越明显。 但真正影响切换性能的其实并不是这关键性的上下文切换代码，而是切换之后可能带来的cache缺失问题！

	+ 不需要多线程的锁机制，因为只有一个线程，也不存在同时写变量冲突，在协程中控制共享资源不加锁，只需要判断状态就好了，所以执行效率比多线程高很多。 


2. libco

	2.1 主要接口

	- co_create： 参数里面有一个函数指针，用于创建一个新协程，可以理解为创建了一个上下文，但只是创建，啥都没干，就pthread_create。

	- co_resume：执行某一个协程，可以看里面的实现，里面有一个pCallStack，这里面装的其实就是协程栈，要执行的时候，就取出当前执行的协程，也就是栈里面最后一个协程，然后对传进来的协程判断一下，做一个初始化，压入栈，然后和当前的协程swap一下。这个swap其实就是在把当前这个线程要执行的下一条指令。

	- co_yield_ct：让某个线程（可以认为一个线程，关联一个stCoRotinueEnv_t）当前正在执行的协程，yield让出时间片（在操作系统里面，yield一般是指主动把CPU时间片让出来，也就是暂停运行，直到被唤醒）。这里可以看到里面的实现其实就是找到当前执行的协程（iCallStackSize - 1，我们称作A）和上一个协程（iCallStackSize - 2，我们称作B），然后swap一下这两个协程，并且注意，这里iCallStackSize -- 了，也就是把这个yield的协程A从协程栈里面去掉了，但是由于这个A协程已经执行了一些代码了，所以在这个A协程的上下文中也保存了执行的进度，如果下次这个A被co_resume，会从A停止的地方开始执行，而不是重新开始，所以co_resume的名字叫作resume（resume：继续的意思）。

	2.2 实现原理

	2.2.1 主要技术

	- 协程切换：libco自己实现了切换函数上下文的函数（coctx_swap，通过修改存在CPU寄存器的栈指针以及函数，函数参数指针）

	- Hook系统函数：通过hook系统函数，应用层依然可以同步调用send,recv, poll等系统函数，但是底层通过加入epoll，主动让出当前CPU，实现了异步IO的方式。

	- epoll+时间轮：管理网络事件以及超时事件

	- 共享栈模式：节约系统内存，从而支持更多的连接,共享栈切换时需要保存栈内容

	2.2.2 代码与数据结构

	+ struct stCoRoutineEnv_t 协程管理类
	+ struct stCoRoutine_t 协程类

	2.2.3 协程切换原理
	
	说到切换，最关键的函数便是`void coctx_swap( coctx_t *,coctx_t* ) asm("coctx_swap");`
	两个参数都是 coctx_t *指针类型，其中第一个参数表示要切出的协程，第二个参数表示切出后要进入的协程。 在进入 coctx_swap 时，第一个参数值已经放到了 %rdi 寄存器中，第二个参数值已经放到了 %rsi 寄存器中，并且栈指针 %rsp 指向的位置即栈顶中存储的是父函数的返回地址。进入 coctx_swap 后，堆栈的状态如下： 
	![avatar](https://github.com/ranchochen/libco-1/blob/master/notes/libco_swap_stack.JPG)
	
	由于coctx_swap 是在 co_swap() 函数中调用的，下面所提及的协程的返回地址就是 co_swap() 中调用 coctx_swap() 之后下一条指令的地址:

	    void co_swap(stCoRoutine_t* curr, stCoRoutine_t* pending_co) {
    	 ....
    	// 从本协程切出
    	coctx_swap(&(curr->ctx),&(pending_co->ctx) );
    
    	// 此处是返回地址，即协程恢复时开始执行的位置
    	stCoRoutineEnv_t* curr_env = co_get_curr_thread_env();
    	....
    	}
		
		.........................
		coctx_swap:
		   leaq 8(%rsp),%rax
		   leaq 112(%rdi),%rsp
		   pushq %rax
		   pushq %rbx
		   pushq %rcx
		   pushq %rdx
		
		   pushq -8(%rax) //ret func addr
		
		   pushq %rsi
		   pushq %rdi
		   pushq %rbp
		   pushq %r8
		   pushq %r9
		   pushq %r12
		   pushq %r13
		   pushq %r14
		   pushq %r15
		
		   movq %rsi, %rsp
		   popq %r15
		   popq %r14
		   popq %r13
		   popq %r12
		   popq %r9
		   popq %r8
		   popq %rbp
		   popq %rdi
		   popq %rsi
		   popq %rax //ret func addr
		   popq %rdx
		   popq %rcx
		   popq %rbx
		   popq %rsp
		   pushq %rax
		
		   xorl %eax, %eax
		   ret
	
	可以看出，coctx_swap 中并未像常规被调用函数一样创立新的栈帧。先看前两条语句：

	       leaq 8(%rsp),%rax
    	   leaq 112(%rdi),%rsp

	leaq 用于把其第一个参数的值赋值给第二个寄存器参数。第一条语句用来把 8(%rsp) 的本身的值存入到 %rax 中，注意这里使用的并不是 8(%rsp) 指向的值，而是把 8(%rsp) 表示的地址赋值给了 %rax。这一地址是父函数栈帧中除返回地址外栈帧顶的位置。

	在第二条语句 leaq 112(%rdi), %rsp 中，%rdi 存放的是coctx_swap 第一个参数的值，这一参数是指向 coctx_t 类型的指针，表示当前要切出的协程，这一类型的定义如下：

		struct coctx_t {
		void *regs[ 14 ]; 
		size_t ss_size;
		char *ss_sp;
		
		};
	
	因而 112(%rdi) 表示的就是第一个协程的 coctx_t 中 regs[14] 数组的下一个64位地址。而接下来的语句：

		pushq %rax   
		pushq %rbx
		pushq %rcx
		pushq %rdx
		pushq -8(%rax) //ret func addr
		pushq %rsi
		pushq %rdi
		pushq %rbp
		pushq %r8
		pushq %r9
		pushq %r12
		pushq %r13
		pushq %r14
		pushq %r15

	第一条语句 pushq %rax 用于把 %rax 的值放入到 regs[13] 中，resg[13] 用来存储第一个协程的 %rsp 的值。这时 %rax 中的值是第一个协程 coctx_swap 父函数栈帧除返回地址外栈帧顶的地址。由于 regs[] 中有单独的元素存储返回地址，栈中再保存返回地址是无意义的，因而把父栈帧中除返回地址外的栈帧顶作为要保存的 %rsp 值是合理的。当协程恢复时，把保存的 regs[13] 的值赋值给 %rsp 即可恢复本协程 coctx_swap 父函数堆栈指针的位置。第一条语句之后的语句就是用pushq 把各CPU 寄存器的值依次从 regs 尾部向前压入。即通过调整%rsp 把 regs[14] 当作堆栈，然后利用 pushq 把寄存器的值和返回地址存储到 regs[14] 整个数组中。regs[14] 数组中各元素与其要存储的寄存器对应关系如下：

		//-------------
		// 64 bit
		//low | regs[0]: r15 |
		//| regs[1]: r14 |
		//| regs[2]: r13 |
		//| regs[3]: r12 |
		//| regs[4]: r9  |
		//| regs[5]: r8  | 
		//| regs[6]: rbp |
		//| regs[7]: rdi |
		//| regs[8]: rsi |
		//| regs[9]: ret |  //ret func addr, 对应 rax
		//| regs[10]: rdx |
		//| regs[11]: rcx | 
		//| regs[12]: rbx |
		//hig | regs[13]: rsp |

	接下来的汇编语句：

		movq %rsi, %rsp
		popq %r15
		popq %r14
		popq %r13
		popq %r12
		popq %r9
		popq %r8
		popq %rbp
		popq %rdi
		popq %rsi
		popq %rax //ret func addr
		popq %rdx
		popq %rcx
		popq %rbx
		popq %rsp   

	这里用的方法还是通过改变%rsp 的值，把某块内存当作栈来使用。第一句 movq %rsi, %rsp 就是让%rsp 指向 coctx_swap 第二个参数，这一参数表示要进入的协程。而第二个参数也是coctx_t 类型的指针，即执行完 movq 语句后，%rsp 指向了第二个参数 coctx_t 中 regs[0]，而之后的pop 语句就是用 regs[0-13] 中的值填充cpu 的寄存器，这里需要注意的是popq 会使得 %rsp 的值增加而不是减少，这一点保证了会从 regs[0] 到regs[13] 依次弹出到 cpu 寄存器中。在执行完最后一句 popq %rsp 后，%rsp 已经指向了新协程要恢复的栈指针（即新协程之前调用 coctx_swap 时父函数的栈帧顶指针），由于每个协程都有一个自己的栈空间，可以认为这一语句使得%rsp 指向了要进入协程的栈空间。

	coctx_swap 中最后三条语句如下：

    	pushq %rax
    	xorl %eax, %eax
    	ret

	pushq %rax 用来把 %rax 的值压入到新协程的栈中，这时 %rax 是要进入的目标协程的返回地址，即要恢复的执行点。然后用 xorl 把 %rax 低32位清0以实现地址对齐。最后ret 语句用来弹出栈的内容，并跳转到弹出的内容表示的地址处，而弹出的内容正好是上面 pushq %rax 时压入的 %rax 的值，即之前保存的此协程的返回地址。即最后这三条语句实现了转移到新协程返回地址处执行，从而完成了两个协程的切换。可以看出，这里通过调整%rsp 的值来恢复新协程的栈，并利用了 ret 语句来实现修改指令寄存器 %rip 的目的，通过修改 %rip 来实现程序运行逻辑跳转。注意%rip 的值不能直接修改，只能通过 call 或 ret 之类的指令来间接修改。

	整体上看来，协程的切换其实就是cpu 寄存器内容特别是%rip 和 %rsp 的写入和恢复，因为cpu 的寄存器决定了程序从哪里执行（%rip) 和使用哪个地址作为堆栈 （%rsp）。寄存器的写入和恢复为：B regs[14]->cpu->A regs[14]。

	执行完，就将之前 cpu 寄存器的值保存到了协程A 的 regs[14] 中，而将协程B regs[14] 的内容写入到了寄存器中，从而使执行逻辑跳转到了 B 协程 regs[14] 中保存的返回地址处开始执行，即实现了协程的切换（从A 协程切换到了B协程执行）。



3. 参考资料
> 1. https://www.liaoxuefeng.com/wiki/001374738125095c955c1e6d8bb493182103fac9270762a000/0013868328689835ecd883d910145dfa8227b539725e5ed000
> 2. https://www.cnblogs.com/unnamedfish/p/8460432.html
> 3. https://blog.csdn.net/u011579138/article/details/81839840
> 4. https://blog.csdn.net/cjk_cynosure/article/details/80139106
> 5. https://blog.csdn.net/lqt641/article/details/73287231
> 6. https://www.zhihu.com/question/52193579/answer/129597362