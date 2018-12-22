---
title: 分析spice-gtk中C语言的协程实现
descriptions: 用C语言的协程实现理解python中协程的各种机制
categories: glib
tags: GObject
---

先来看两组函数

## getcontext, setcontext

### NAME

getcontext, setcontext - 获取和设置用户上下文

### 摘要

```c
#include <ucontext.h>

int getcontext(ucontext_t *ucp);
int setcontext(const ucontext_t *ucp);
```

### 描述

在一个类System V环境，如果它在<ucontext.h>中定义了类型ucontext_t，并且有getcontext, setcontext, makecontext 和swapcontext 这四个函数， 那么它就允许在一个进程内的多个控制线程之间进行用户级上下文切换。

`mcontext_t` 类型是机器相关的且不透明，`ucontext_t`是一个至少具有以下字段的结构体

```c
typedef struct ucontext_t {
   struct ucontext_t *uc_link;
   sigset_t          uc_sigmask;
   stack_t           uc_stack;
   mcontext_t        uc_mcontext;
   ...
} ucontext_t;
```
`sigset_t` 和 `stack_t` 定义在<signal.h>。   
这里`uc_link`指向当前上下文终止时将恢复的上下文（如果当前上下文是使用makecontext（3）创建的），`uc_sigmask`是在此上下文中被阻塞的信号集（请参阅sigprocmask（2）），`uc_stack`是此上下文使用的堆栈（请参阅sigaltstack（2）），`uc_mcontext`是已保存上下文的特定于机器的表示，包括调用线程的机器寄存器。

函数getcontext（）将ucp指向的结构初始化为当前活动的上下文。

函数setcontext（）恢复ucp指向的用户上下文。 调用成功的话不会返回。 上下文应该通过调用getcontext（）或makecontext（3）获得，或者作为第三个参数传递给信号处理程序。

如果上下文是通过调用getcontext（）获得的，该上下文激活时程序继续执行，就像刚刚返回此调用一样。

如果上下文是通过调用makecontext（3）获得的，该上下文激活时，程序调用函数func继续执行。func是传递给makecontext（3）的第二个参数。 当函数func返回时，使用结构ucp的`uc_link`成员继续执行，该成员由传递给makecontext（3）的第一个参数指定。当此成员为NULL时，线程退出。

如果上下文通过调用信号处理程序获得，那么旧的标准文本表示“程序使用被信号中断的指令之后的指令继续执行”。 但是，在SUSv2中删除了这句话，目前的判决是“结果未指明”

### 返回值

成功时，getcontext（）返回0并且setcontext（）不返回。 出错时，都返回-1并正确设置errno。

### 建议

SUSv2，POSIX.1-2001 ，POSIX.1-2008删除了getcontext()的规范，引用了可移植性问题，并建议重写应用程序以使用POSIX线程代替

### 注意事项

这种机制最早的化身是setjmp（3）/ longjmp（3）机制。 由于这没有定义信号上下文的处理，下一阶段是sigsetjmp（3）/ siglongjmp（3）对。 目前的机制提供了更多的控制。 另一方面，没有简单的方法来检测getcontext（）的返回是来自第一次调用还是来自setcontext（）调用。 用户必须发明自己的簿记设备，并且由于寄存器被恢复，寄存器变量将不起作用。

当信号出现时，将保存当前用户上下文，并由内核为信号处理程序创建新的上下文。 不要使用longjmp（3）离开处理程序：这未定义上下文会发生什么。 请改用siglongjmp（3）或setcontext（）。

## makecontext, swapcontext

### 名称

makecontext, swapcontext  用于操作用户上下文

### 摘要

```c
#include <ucontext.h>

void makecontext(ucontext_t *ucp, void (*func)(), int argc, ...);
int  swapcontext(ucontext_t *oucp, const ucontext_t *ucp);
```

### 描述

在一个类System V环境，如果它在<ucontext.h>中定义了类型ucontext_t，并且有getcontext, setcontext, makecontext 和swapcontext 这四个函数， 那么它就允许在一个进程内的多个控制线程之间进行用户级上下文切换。

对于类型ucontext_t 和前两个函数，参考前文。

makecontext（）函数修改ucp指向的上下文（该上下文通过调用getcontext（3）获得）。 在调用makecontext（）之前，调用者必须为此上下文分配一个新堆栈，并将其地址赋值给`ucp->uc_stack`，定义一个**后继上下文**并将其地址赋值给`ucp->uc_link`。

当稍后激活此上下文时（使用setcontext（3）或swapcontext（）），将调用函数func，并在argc后面传递一系列整数（int）参数; 调用者必须在argc中指定这些参数的数量。当此函数返回时，将激活**后继上下文**。 如果后继上下文指针为NULL，则线程退出。

swapcontext（）函数将当前上下文保存在oucp指向的结构中，然后激活ucp指向的上下文。

### 返回值

成功时，swapcontext（）不会返回。 （但是我们可能会稍后返回，如果oucp被激活，在这种情况下，看起来swapcontext（）会返回0.）出错时，swapcontext（）返回-1并正确设置errno。

### ERRORS

ENOMEM 剩余的堆栈空间不足。

### versions

makecontext() 和 swapcontext() 在glibc 2.1版本后被提供

### 建议

SUSv2，POSIX.1-2001 ，POSIX.1-2008删除了makecontext（）和swapcontext（）的规范，引用了可移植性问题，并建议重写应用程序以使用POSIX线程代替

### 注意事项

`ucp->uc_stack`的解释与sigaltstack（2）中的解释一样，
即，该结构包含要用作堆栈的存储区的开始和长度，而不管堆栈的增长方向如何。 因此，用户程序不必担心这个方向

在int和指针类型大小相同的架构上（例如，x86-32，两种类型都是32位），您可以通过在argc后面将指针作为参数传递给makecontext（）。 但是，这样做不保证是可移植的，根据标准是未定义的，并且不适用于指针大于整数的架构。然而，从版本2.8开始，glibc对makecontext（）进行了一些更改，以允许在某些64位体系结构（例如，x86-64）上这样使用。

### EXAMPLE

下面的例子阐释了getcontext, makecontext, swapcontext 的使用，运行程序产生以下输出

```
$ ./a.out
main: swapcontext(&uctx_main, &uctx_func2)
func2: started
func2: swapcontext(&uctx_func2, &uctx_func1)
func1: started
func1: swapcontext(&uctx_func1, &uctx_func2)
func2: returning
func1: returning
main: exiting
```

程序源码:

```c
#include <ucontext.h>
#include <stdio.h>
#include <stdlib.h>

static ucontext_t uctx_main, uctx_func1, uctx_func2;

#define handle_error(msg) \
   do { perror(msg); exit(EXIT_FAILURE); } while (0)

static void
func1(void)
{
   printf("func1: started\n");
   printf("func1: swapcontext(&uctx_func1, &uctx_func2)\n");
   if (swapcontext(&uctx_func1, &uctx_func2) == -1)
       handle_error("swapcontext");
   printf("func1: returning\n");
}

static void
func2(void)
{
   printf("func2: started\n");
   printf("func2: swapcontext(&uctx_func2, &uctx_func1)\n");
   if (swapcontext(&uctx_func2, &uctx_func1) == -1)
       handle_error("swapcontext");
   printf("func2: returning\n");
}

int
main(int argc, char *argv[])
{
   char func1_stack[16384];
   char func2_stack[16384];

   if (getcontext(&uctx_func1) == -1)
       handle_error("getcontext");
   uctx_func1.uc_stack.ss_sp = func1_stack;
   uctx_func1.uc_stack.ss_size = sizeof(func1_stack);
   uctx_func1.uc_link = &uctx_main;
   makecontext(&uctx_func1, func1, 0);

   if (getcontext(&uctx_func2) == -1)
       handle_error("getcontext");
   uctx_func2.uc_stack.ss_sp = func2_stack;
   uctx_func2.uc_stack.ss_size = sizeof(func2_stack);
   /* Successor context is f1(), unless argc > 1 */
   uctx_func2.uc_link = (argc > 1) ? NULL : &uctx_func1;
   makecontext(&uctx_func2, func2, 0);

   printf("main: swapcontext(&uctx_main, &uctx_func2)\n");
   if (swapcontext(&uctx_main, &uctx_func2) == -1)
       handle_error("swapcontext");

   printf("main: exiting\n");
   exit(EXIT_SUCCESS);
}
```

## 结合实例分析 spice-gtk 中的协程封装

这里只考虑 WITH_UCONTEXT 使用上下文切换的实现

coroutine.h

```c
#ifndef _COROUTINE_H_
#define _COROUTINE_H_

#include "config.h"

#if WITH_UCONTEXT
#include "continuation.h"
#elif WITH_WINFIBER
#include <windows.h>
#else
#include <glib.h>
#endif

struct coroutine
{
	size_t stack_size;
	void *(*entry)(void *);
	int (*release)(struct coroutine *);

	/* read-only */
	int exited;

	/* private */
	struct coroutine *caller;
	void *data;

#if WITH_UCONTEXT
	struct continuation cc;
#elif WITH_WINFIBER
    LPVOID fiber;
    int ret;
#else
	GThread *thread;
	gboolean runnable;
#endif
};

void coroutine_init(struct coroutine *co);

int coroutine_release(struct coroutine *co);

void *coroutine_swap(struct coroutine *from, struct coroutine *to, void *arg);

struct coroutine *coroutine_self(void);

void *coroutine_yieldto(struct coroutine *to, void *arg);

void *coroutine_yield(void *arg);

gboolean coroutine_is_main(struct coroutine *co);

static inline gboolean coroutine_self_is_main(void) {
	return coroutine_self() == NULL || coroutine_is_main(coroutine_self());
}

#endif
```
continuation.h

```c
#ifndef _CONTINUATION_H_
#define _CONTINUATION_H_

#include <stddef.h>
#include <ucontext.h>
#include <setjmp.h>

struct continuation
{
	char *stack;
	size_t stack_size;
	void (*entry)(struct continuation *cc);
	int (*release)(struct continuation *cc);

	/* private */
	ucontext_t uc;
	ucontext_t last;
	int exited;
	jmp_buf jmp;
};

void cc_init(struct continuation *cc);

int cc_release(struct continuation *cc);

/* you can use an uninitialized struct continuation for from if you do not have
   the current continuation handy. */
int cc_swap(struct continuation *from, struct continuation *to);

#define offset_of(type, member) ((unsigned long)(&((type *)0)->member))
#define container_of(obj, type, member) \
        (type *)(((char *)obj) - offset_of(type, member))

#endif
```
continuation.c

```c
#include <config.h>

/* keep this above system headers, but below config.h */
#ifdef _FORTIFY_SOURCE
#undef _FORTIFY_SOURCE
#endif

#include <errno.h>
#include <glib.h>

#include "continuation.h"

/*
 * va_args to makecontext() must be type 'int', so passing
 * the pointer we need may require several int args. This
 * union is a quick hack to let us do that
 */
union cc_arg {
	void *p;
	int i[2];
};

static void continuation_trampoline(int i0, int i1)
{
	union cc_arg arg;
	struct continuation *cc;
	arg.i[0] = i0;
	arg.i[1] = i1;
	cc = arg.p;

	if (_setjmp(cc->jmp) == 0) {
		ucontext_t tmp;
		swapcontext(&tmp, &cc->last);
	}

	cc->entry(cc);
}

void cc_init(struct continuation *cc)
{
	volatile union cc_arg arg;
	arg.p = cc;
	if (getcontext(&cc->uc) == -1)
		g_error("getcontext() failed: %s", g_strerror(errno));
	cc->uc.uc_link = &cc->last;
	cc->uc.uc_stack.ss_sp = cc->stack;
	cc->uc.uc_stack.ss_size = cc->stack_size;
	cc->uc.uc_stack.ss_flags = 0;

	makecontext(&cc->uc, (void *)continuation_trampoline, 2, arg.i[0], arg.i[1]);
	swapcontext(&cc->last, &cc->uc);
}

int cc_release(struct continuation *cc)
{
	if (cc->release)
		return cc->release(cc);

	return 0;
}

int cc_swap(struct continuation *from, struct continuation *to)
{
	to->exited = 0;
	if (getcontext(&to->last) == -1)
		return -1;
	else if (to->exited == 0)
		to->exited = 1; // so when coroutine finishes
    else if (to->exited == 1)
            return 1; // it ends up here

	if (_setjmp(from->jmp) == 0)
		_longjmp(to->jmp, 1);

	return 0;
}
```
coroutine_ucontext.c

```c
#include <config.h>
#include <glib.h>

#ifdef HAVE_SYS_TYPES_H
#include <sys/types.h>
#endif
#include <sys/mman.h>
#include <errno.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include "coroutine.h"

#ifndef MAP_ANONYMOUS
# define MAP_ANONYMOUS MAP_ANON
#endif

int coroutine_release(struct coroutine *co)
{
	return cc_release(&co->cc);
}

static int _coroutine_release(struct continuation *cc)
{
	struct coroutine *co = container_of(cc, struct coroutine, cc);

	if (co->release) {
		int ret = co->release(co);
		if (ret < 0)
			return ret;
	}

	munmap(co->cc.stack, co->cc.stack_size);

	co->caller = NULL;

	return 0;
}

static void coroutine_trampoline(struct continuation *cc)
{
	struct coroutine *co = container_of(cc, struct coroutine, cc);
	co->data = co->entry(co->data);
}

void coroutine_init(struct coroutine *co)
{
	if (co->stack_size == 0)
		co->stack_size = 16 << 20;

	co->cc.stack_size = co->stack_size;
	co->cc.stack = mmap(0, co->stack_size,
			    PROT_READ | PROT_WRITE,
			    MAP_PRIVATE | MAP_ANONYMOUS,
			    -1, 0);
	if (co->cc.stack == MAP_FAILED)
		g_error("mmap(%" G_GSIZE_FORMAT ") failed: %s",
			co->stack_size, g_strerror(errno));

	co->cc.entry = coroutine_trampoline;
	co->cc.release = _coroutine_release;
	co->exited = 0;

	cc_init(&co->cc);
}

#if 0
static __thread struct coroutine leader;
static __thread struct coroutine *current;
#else
static struct coroutine leader; //leader 表示主协程上下文
static struct coroutine *current;
#endif

struct coroutine *coroutine_self(void)
{
	if (current == NULL)
		current = &leader;
	return current;
}

void *coroutine_swap(struct coroutine *from, struct coroutine *to, void *arg)
{
	int ret;
	to->data = arg;
	current = to;
	ret = cc_swap(&from->cc, &to->cc);
	if (ret == 0)
		return from->data;
	else if (ret == 1) {
		coroutine_release(to);
		current = from;
		to->exited = 1;
		return to->data;
	}

	return NULL;
}

void *coroutine_yieldto(struct coroutine *to, void *arg)
{
	g_return_val_if_fail(!to->caller, NULL);
	g_return_val_if_fail(!to->exited, NULL);

	to->caller = coroutine_self();
	return coroutine_swap(coroutine_self(), to, arg);
}

void *coroutine_yield(void *arg)
{
	struct coroutine *to = coroutine_self()->caller;
	if (!to) {
		fprintf(stderr, "Co-routine is yielding to no one\n");
		abort();
	}
	coroutine_self()->caller = NULL;
	return coroutine_swap(coroutine_self(), to, arg);
}

gboolean coroutine_is_main(struct coroutine *co)
{
    return (co == &leader);
}
```
coroutine.c

```c
#include <glib.h>
#include <stdio.h>
#include <string.h>
#include <stdlib.h>

#include "coroutine.h"

static gpointer co_entry_check_self(gpointer data)
{
    g_assert(data == coroutine_self());
    g_assert(!coroutine_self_is_main());

    return NULL;
}

static gpointer co_entry_42(gpointer data)
{
    g_assert(GPOINTER_TO_INT(data) == 42);
    g_assert(!coroutine_self_is_main());

    return GINT_TO_POINTER(0x42);
}

static void test_coroutine_simple(void)
{
    struct coroutine *self = coroutine_self();
    struct coroutine co = {
        .stack_size = 16 << 20,
        .entry = co_entry_42,
    };
    gpointer result;

    g_assert(coroutine_self_is_main());

    coroutine_init(&co);
    result = coroutine_yieldto(&co, GINT_TO_POINTER(42));
    g_assert_cmpint(GPOINTER_TO_INT(result), ==, 0x42);

#if GLIB_CHECK_VERSION(2,34,0)
    g_test_expect_message(G_LOG_DOMAIN, G_LOG_LEVEL_CRITICAL, "*!to->exited*");
    coroutine_yieldto(&co, GINT_TO_POINTER(42));
    g_test_assert_expected_messages();
#endif

    g_assert(self == coroutine_self());
    g_assert(coroutine_self_is_main());
}

static gpointer co_entry_two(gpointer data)
{
    struct coroutine *self = coroutine_self();
    struct coroutine co = {
        .stack_size = 16 << 20,
        .entry = co_entry_check_self,
    };

    g_assert(!coroutine_self_is_main());
    coroutine_init(&co);
    coroutine_yieldto(&co, &co);

    g_assert(self == coroutine_self());
    return NULL;
}

static void test_coroutine_two(void)
{
    struct coroutine *self = coroutine_self();
    struct coroutine co = {
        .stack_size = 16 << 20,
        .entry = co_entry_two,
    };

    coroutine_init(&co);
    coroutine_yieldto(&co, NULL);

    g_assert(self == coroutine_self());
}

static gpointer co_entry_yield(gpointer data)
{
    gpointer val;

    g_assert(data == NULL);
    val = coroutine_yield(GINT_TO_POINTER(1));
    g_assert_cmpint(GPOINTER_TO_INT(val), ==, 2);

    g_assert(!coroutine_self_is_main());

    val = coroutine_yield(GINT_TO_POINTER(3));
    g_assert_cmpint(GPOINTER_TO_INT(val), ==, 4);

    return NULL;
}

static void test_coroutine_yield(void)
{
    struct coroutine *self = coroutine_self();
    struct coroutine co = {
        .stack_size = 16 << 20,
        .entry = co_entry_yield,
    };
    gpointer val;

    coroutine_init(&co);
    val = coroutine_yieldto(&co, NULL);

    g_assert(self == coroutine_self());
    g_assert_cmpint(GPOINTER_TO_INT(val), ==, 1);

    val = coroutine_yieldto(&co, GINT_TO_POINTER(2));

    g_assert(self == coroutine_self());
    g_assert_cmpint(GPOINTER_TO_INT(val), ==, 3);

    val = coroutine_yieldto(&co, GINT_TO_POINTER(4));

    g_assert(self == coroutine_self());
    g_assert(val == NULL);

#if GLIB_CHECK_VERSION(2,34,0)
    g_test_expect_message(G_LOG_DOMAIN, G_LOG_LEVEL_CRITICAL, "*!to->exited*");
    coroutine_yieldto(&co, GINT_TO_POINTER(42));
    g_test_assert_expected_messages();
#endif
}

int main(int argc, char* argv[])
{
    g_test_init(&argc, &argv, NULL);

    g_test_add_func("/coroutine/simple", test_coroutine_simple);
    g_test_add_func("/coroutine/two", test_coroutine_two);
    g_test_add_func("/coroutine/yield", test_coroutine_yield);

    return g_test_run ();
}
```

coroutine_ucontext.c 是使用ucontext上下文实现的协程，还有用gthread和winfibers的协程实现。  
coroutine.c是协程的一个使用示例。

### 协程初始化

分析coroutine.c示例，看一下协程是如何初始化的

1. 分配一个coroutine结构体co，设置co的栈大小和入口函数，调用coroutine_init
2. 在`coroutine_init`里面，设置co的成员cc的栈，栈大小，默认的entry和release函数等字段。调用`cc_init`初始化cc
3. 在`cc_init`中，使用getcontext获取当前上下文，makecontext修改上下文，调用swapcontext保存当前上下文到cc->last, 切换到cc->uc上下文，即函数continuation_trampoline。
4. 在`continuation_trampoline`中调用`_setjmp(cc->jmp)`设置跳转点，再切回cc->last，即`cc_init`函数返回，`coroutine_init`初始化结束。

### 协程切换

使用`coroutine_yieldto` 切换到协程执行，来看看`coroutine_yieldto` 是如何实现上下文切换的：

1. 设置to->caller协程为当前协程, 调用coroutine_swap()函数执行切换。
2. 在`coroutine_swap`中设置current协程为to，参数arg赋值给to->data，调用`cc_swap`执行真正的上下文切换
3. 在`cc_swap`中，首先设置to->last上下文以便协程函数执行完后切换回来，然后调用`_setjmp(from->jmp)`设置当前上下文的跳转点，并调用`_longjmp(to->jmp,1)`跳转到to协程的跳转点，此跳转点及to协程初始化时在`continuation_trampoline`中调用`_setjmp`设置的, 即初始化中的第4步。
4. 在`continuation_trampoline`函数中执行cc->entry(cc), 即`coroutine_trampoline`
5. 在`coroutine_trampoline` 中执行co->entry(co->data),即初始化中第一步的入口函数。并将结果赋值到co->data。
6. co->entry执行结束后，执行to->last, 此即第3步中获取上下文的地方`cc_swap`。
7. `cc_swap` 中从getcontext返回后，此时to->exited为1，因此cc_swap 返回1
8. 在`coroutine_swap`中释放to协程，设置current为当前协程from，返回co->entry的执行结果co->data, 此即`coroutine_yieldto`的执行结果。

### 协程传值

重点看coroutine.c中执行的第三个示例，主协程调用`coroutine_yieldto`切换到协程co执行，co调用`coroutine_yield`切换回主协程，`coroutine_yield`的参数值作为`coroutine_yieldto`的返回值返回，主协程继续执行。当主协程再次调用`coroutine_yieldto`切换到co时，其参数值作为co的`coroutine_yield`的返回值返回，co继续执行。那么主协程的`coroutine_yieldto` 和co协程的`coroutine_yield` 是如何实现传值的呢？换句话说，为什么`coroutine_yield`的参数值会作为`coroutine_yieldto`的返回值返回？反之亦然。下面来仔细分析一下。

分便于描述，主协程使用leader表示，并跳过协程切换的一些无关步骤：

1. leader调用`coroutine_yieldto`切换到co时，在`coroutine_swap`中将参数值arg赋值给co->data，调用`cc_swap`时，执行`_setjmp(leader->jmp)`, `_longjmp(co->jmp, 1)`切换到co运行，此处from就是leader， to就是co。
2. co调用`coroutine_yield`切换回leader时，同样会在`coroutine_swap`中将arg赋值给leader->data，然后在`cc_swap`中，执行`_setjmp(co->jmp)`, `_longjmp(leader->jmp, 1)`切换回leader，即第1步中leader在`cc_swap`中调用`_setjmp(leader->jmp)`设置的跳转点，`leader的cc_swap`返回0，`coroutine_swap` 返回leader->data(leader上下文中的from就是leader), 这也是`coroutine_yieldto`的返回值，此即co调用`coroutine_yield`的参数值。
3. leader再次调用`coroutine_yieldto` 切换到co，将参数值arg赋值给co->data，在`cc_swap`中执行`_setjmp(leader->jmp)`, `_longjmp(co->jmp, 1)`切换到co协程，即跳转到第2步中`_setjmp(co->jmp)`设置的跳转点，使co的`cc_swap`返回0，从而`coroutine_yield`返回co->data(co上下文中的from是co)，此即leader调用coroutine_yieldto的参数值。

通过上面的步骤，`coroutine_yieldto` 和`coroutine_yield`之间实现了传值。  
要理解上面的逻辑，关键是要理解上下文切换时，代码中from和to指代的协程是会变化的。即leader上下文中from是leader，to是co；co上下文中from是co，to是leader。

## 结论

通过上面的分析可以得出结论：协程是用户级的上下文切换，协程是同步的。

## 参考

+ [getcontext](http://man7.org/linux/man-pages/man3/getcontext.3.html)
+ [makecontext](http://man7.org/linux/man-pages/man3/makecontext.3.html)
+ [spice-gtk](https://github.com/freedesktop/spice-gtk)