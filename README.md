# Task Management

In the previous section, I specified that a task is either an infinite loop function or a function that deletes itself when it is done executing. Note that the task code is not actually deleted — µCOS-II simply doesn’t know about the task anymore, so that code will not run. <mark style="color:blue;">A task looks just like any other C function, containing a return type and an argument, but it must never return. The return type of a task must always be declared void.</mark> The functions described in this chapter are found in the file `OS_TASK.C`. To review, a task must have one of the two structures:

```c
void YourTask (void *pdata) 
{ 
    for (;;) { 
        /* USER CODE */ 
        Call one of uC/OS-II's services:
            OSFlagPend();
            OSMboxPend();
            OSMutexPend();
            OSQPend();
            OSSemPend();
            OSTaskSuspend(OS_PRIO_SELF); 
            OSTimeDly();
            OSTimeDlyHMSM();
        /* USER CODE */
    } 
} 
or,
void YourTask (void *pdata)
{ 
    /* USER CODE */ 
    OSTaskDel(OS_PRIO_SELF);
}

```

This chapter describes the services that allow your application to create a task, delete a task, change a task’s priority, suspend and resume a task, and allow your application to obtain information about a task.

µCOS-II can manage up to <mark style="color:blue;">64 tasks</mark>, although <mark style="color:blue;">µCOS-II reserves the four highest priority tasks and the four lowest priority tasks for its own use</mark>. However, at this time, only two priority levels are actually used by µCOS-II: `OSTaskCreate` and `OS_LOWEST_PRIO-1` (see `OS_CFG.H`). This leaves you with up to 56 application tasks. The lower the value of the priority, the higher the priority of the task. In the current version of µCOS-II, the task priority number also serves as the task identifier.

* [Creating a Task, OSTaskCreate()](https://micrium.atlassian.net/wiki/spaces/osiidoc/pages/163900/Task+Management#TaskManagement-CreatingaTask%2COSTaskCreate\(\))
* [Creating a Task, OSTaskCreateExt()](https://micrium.atlassian.net/wiki/spaces/osiidoc/pages/163900/Task+Management#TaskManagement-CreatingaTask%2COSTaskCreateExt\(\))
* [Task Stacks](https://micrium.atlassian.net/wiki/spaces/osiidoc/pages/163900/Task+Management#TaskManagement-TaskStacks)
* [Stack Checking, OSTaskStkChk()](https://micrium.atlassian.net/wiki/spaces/osiidoc/pages/163900/Task+Management#TaskManagement-StackChecking%2COSTaskStkChk\(\))
* [Deleting a Task, OSTaskDel()](https://micrium.atlassian.net/wiki/spaces/osiidoc/pages/163900/Task+Management#TaskManagement-DeletingaTask%2COSTaskDel\(\))
* [Requesting to Delete a Task, OSTaskDelReq()](https://micrium.atlassian.net/wiki/spaces/osiidoc/pages/163900/Task+Management#TaskManagement-RequestingtoDeleteaTask%2COSTaskDelReq\(\))
* [Changing a Task’s Priority, OSTaskChangePrio()](https://micrium.atlassian.net/wiki/spaces/osiidoc/pages/163900/Task+Management#TaskManagement-ChangingaTask%E2%80%99sPriority%2COSTaskChangePrio\(\))
* [Suspending a Task, OSTaskSuspend()](https://micrium.atlassian.net/wiki/spaces/osiidoc/pages/163900/Task+Management#TaskManagement-SuspendingaTask%2COSTaskSuspend\(\))
* [Resuming a Task, OSTaskResume()](https://micrium.atlassian.net/wiki/spaces/osiidoc/pages/163900/Task+Management#TaskManagement-ResumingaTask%2COSTaskResume\(\))
* [Getting Information about a Task, OSTaskQuery()](https://micrium.atlassian.net/wiki/spaces/osiidoc/pages/163900/Task+Management#TaskManagement-GettingInformationaboutaTask%2COSTaskQuery\(\))

### Creating a Task, OSTaskCreate() <a href="#taskmanagement-creatingatask-ostaskcreate" id="taskmanagement-creatingatask-ostaskcreate"></a>

In order for µCOS-II to manage your task, you must create it. You create a task by passing its address and other arguments to one of two functions: `OSTaskCreate()` or `OSTaskCreateExt()`. `OSTaskCreate()` is backward compatible with µCOS, and `OSTaskCreateExt()` is an extended version of `OSTaskCreate()`, providing additional features. A task can be created using either function. A task can be created prior to the start of multitasking or by another task. You must create at least one task before you start multitasking \[i.e., before you call `OSStart()`]. <mark style="color:blue;">A task cannot be created by an ISR(interrupt service routine).</mark>

The code for `OSTaskCreate()` is shown in Listing 4.1. As can be seen, `OSTaskCreate()` requires four arguments. task is a pointer to the task code, pdata is a pointer to an argument that is passed to your task when it starts executing, ptos is a pointer to the top of the stack that is assigned to the task (see section 4.02, Task Stacks), and prio is the desired task priority.

{% code title="Listing 4.1" %}
```c
INT8U OSTaskCreate (void (*task)(void *pd), void *pdata, OS_STK *ptos, INT8U prio)
{
#if OS_CRITICAL_METHOD == 3
    OS_CPU_SR  cpu_sr;
#endif
    void      *psp;
    INT8U      err;
 
#if OS_ARG_CHK_EN > 0u
    if (prio > OS_LOWEST_PRIO) {                                           (1)
        return (OS_ERR_PRIO_INVALID);
    }
#endif
    OS_ENTER_CRITICAL();
    if (OSIntNesting > 0u) {                 
        OS_EXIT_CRITICAL();
        return (OS_ERR_TASK_CREATE_ISR);
    }

    if (OSTCBPrioTbl[prio] == (OS_TCB *)0) {                               (2)
        OSTCBPrioTbl[prio] = (OS_TCB *)OS_TCB_RESERVED;                    (3)
        OS_EXIT_CRITICAL();                                                (4)
        psp = (void *)OSTaskStkInit(task, pdata, ptos, 0);                 (5)
        err = OS_TCBInit(prio, psp, (void *)0, 0, 0, (void *)0, 0);        (6)
        if (err == OS_ERR_NONE) {                                          (7)
            if (OSRunning == OS_TRUE) {                                    (8)
                OS_Sched();                                                (9)
            }
        } else {
            OS_ENTER_CRITICAL();
            OSTCBPrioTbl[prio] = (OS_TCB *)0;                             (10)
            OS_EXIT_CRITICAL();
        }
        return (err);
    }
    OS_EXIT_CRITICAL();
    return (OS_ERR_PRIO_EXIST);
}
```
{% endcode %}

(1) If the configuration constant `OS_ARG_CHK_EN` (see file `OS_CFG.H`) is set to 1, `OSTaskCreate()` checks that the task priority is valid. <mark style="color:blue;">The priority of a task must be a number between 0 and</mark> <mark style="color:blue;"></mark><mark style="color:blue;">`OS_LOWEST_PRIO`</mark><mark style="color:blue;">, inclusive.</mark> Please note that, <mark style="color:purple;">`OS_LOWEST_PRIO`</mark> <mark style="color:purple;"></mark><mark style="color:purple;">is reserved by µCOS-II’s idle task.</mark> Don’t worry, your application will not be able to call `OSTaskCreate()` and create a task at priority `OS_LOWEST_PRIO` because it would have already been ‘reserved’ for the idle task by `OSInit()`. <mark style="color:blue;">In this case,</mark> <mark style="color:blue;"></mark><mark style="color:blue;">`OSTaskCreate()`</mark> <mark style="color:blue;"></mark><mark style="color:blue;">would return OS\_PRIO\_EXIST.</mark>

(2) Next, `OSTaskCreate()` makes sure that a task has not already been created at the desired priority. <mark style="color:purple;">With µCOS-II, all tasks must have a unique priority.</mark>

(3) If the desired priority is free, µCOS-II reserves the priority by placing a non-NULL pointer in `OSTCBPrioTbl[]`.

(4) This allows `OSTaskCreate()` to re-enable interrupts while it sets up the rest of the data structures for the task because no other concurrent calls to `OSTaskCreate()` can now use this priority.

(5) `OSTaskCreate()` then calls `OSTaskStkInit()`, which is responsible for setting up the task stack. This function is processor specific and is found in `OS_CPU_C.C`. Refer to Chapter 13, Porting µCOS-II, for details on how to implement `OSTaskStkInit()`. If you already have a port of µCOS-II for the processor you are intending to use, you don’t need to be concerned about implementation details. `OSTaskStkInit()` returns the new top-of-stack (psp), which will be saved in the task’s `OS_TCB`. You should note that the fourth argument (opt) to `OSTaskStkInit()` is set to 0. This is because, unlike `OSTaskCreateExt()`, `OSTaskCreate()` does not support options, so there are no options to pass to `OSTaskStkInit()`. <mark style="color:blue;">µCOS-II supports processors that have stacks that grow either from high to low memory or from low to high memory.</mark> When you call `OSTaskCreate()`, you must know how the stack grows (see `OS_STACK_GROWTH` in `OS_CPU.H` of the processor you are using) because you must pass the task’s top-of-stack to `OSTaskCreate()`, which can be either the lowest or the highest memory location of the stack.

(6) Once `OSTaskStkInit()` has completed setting up the stack, `OSTaskCreate()` calls `OS_TCBInit()` to obtain and initialize an `OS_TCB` from the pool of free `OS_TCBs`. The code for `OS_TCBInit()` was described in Section 3.?? and is found in `OS_CORE.C` instead of `OS_TASK.C`.

(7) If the stack frame and the task's TCB are properly initialized ...

(8) ... if multitasking has already started then ...

(9) The scheduler is called to determine whether the newly created task has a higher priority than the task that called OSTaskCreate(). Creating a higher priority task results in a context switch to the new task. <mark style="color:blue;">If the task was created before multitasking has started \[i.e., you did not call</mark> <mark style="color:blue;"></mark><mark style="color:blue;">`OSStart()`</mark> <mark style="color:blue;"></mark><mark style="color:blue;">yet], the scheduler is not called.</mark>

(10) If `OS_TCBInit()` failed, the priority level is relinquished(放弃) by setting the entry in `OSTCBPrioTbl[prio]` to 0.

### Creating a Task, OSTaskCreateExt() <a href="#taskmanagement-creatingatask-ostaskcreateext" id="taskmanagement-creatingatask-ostaskcreateext"></a>

Creating a task using `OSTaskCreateExt()` offers more flexibility, but at the expense of additional overhead. The code for `OSTaskCreateExt()` is shown in Listing 4.2.

As can be seen, `OSTaskCreateExt()` requires nine arguments! The first four arguments (task, pdata, ptos, and prio) are exactly the same as in `OSTaskCreate()`, and they are located in the same order. I did this to make it easier to migrate your code to use `OSTaskCreateExt()`.

| **id**        | establishes a unique identifier for the task being created. This argument has been added for future expansion and is otherwise unused by µCOS-II. This identifier will allow me to extend µCOS-II beyond its limit of 64 tasks. For now, simply set the task’s ID to the same value as the task’s priority.                                                                                                                                           |
| ------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **pbos**      | is a pointer to the task’s bottom-of-stack and this argument is used to perform stack checking.                                                                                                                                                                                                                                                                                                                                                       |
| **stk\_size** | specifies the size of the stack in number of elements. This means that if a stack entry is four bytes wide, then a stk\_size of 1000 means that the stack will have 4,000 bytes. Again, this argument is used for stack checking.                                                                                                                                                                                                                     |
| **pext**      | is a pointer to a user-supplied data area that can be used to extend the `OS_TCB` of the task. For example, you can add a name to a task (see Example 3 in Chapter 1), storage for the contents of floating-point registers (see Example 4 in Chapter 1) during a context switch, a port address to trigger an oscilloscope during a context switch, and more.                                                                                        |
| **opt**       | specifies options to `OSTaskCreateExt()`, specifying whether stack checking is allowed, whether the stack will be cleared, whether floating-point operations are performed by the task, etc. `uCOS_II.H` contains a list of available options (`OS_TASK_OPT_STK_CHK`, `OS_TASK_OPT_STK_CLR`, and `OS_TASK_OPT_SAVE_FP`). Each option consists of a bit. The option is selected when the bit is set (simply OR the above `OS_TASK_OPT_???` constants). |

{% code title="Listing 4.2" %}
```c
INT8U  OSTaskCreateExt (void   (*task)(void *pd),
                        void    *pdata,
                        OS_STK  *ptos,
                        INT8U    prio,
                        INT16U   id,
                        OS_STK  *pbos,
                        INT32U   stk_size,
                        void    *pext,
                        INT16U   opt)
{
#if OS_CRITICAL_METHOD == 3                  
    OS_CPU_SR  cpu_sr;
#endif
    OS_STK    *psp;
    INT8U      err;
 
 
#if OS_ARG_CHK_EN > 0
    if (prio > OS_LOWEST_PRIO) {                                   (1)
        return (OS_ERR_PRIO_INVALID);
    }
#endif
    OS_ENTER_CRITICAL();
    if (OSIntNesting > 0u) {                 
        OS_EXIT_CRITICAL();
        return (OS_ERR_TASK_CREATE_ISR);
    }

    if (OSTCBPrioTbl[prio] == (OS_TCB *)0) {                       (2)
        OSTCBPrioTbl[prio] = (OS_TCB *)OS_TCB_RESERVED;            (3)
                                             
        OS_EXIT_CRITICAL();                                        (4)
 
        psp = (OS_STK *)OSTaskStkInit(task, pdata, ptos, opt);      (5)
        err = OS_TCBInit(prio, psp, pbos, id, stk_size, pext, opt); (6)
        if (err == OS_ERR_NONE) {                                   (7)
            if (OSRunning == TRUE) {                                (8)   
                OS_Sched();                                         (9)
            }
        } else {
            OS_ENTER_CRITICAL();
            OSTCBPrioTbl[prio] = (OS_TCB *)0;                      (10) 
            OS_EXIT_CRITICAL();
        }
        return (err);
    }
    OS_EXIT_CRITICAL();
    return (OS_ERR_PRIO_EXIST);
}
```
{% endcode %}

(1) `OSTaskCreateExt()` starts by checking that the task priority is valid. <mark style="color:blue;">The priority of a task must be a number between 0 and</mark> <mark style="color:blue;"></mark><mark style="color:blue;">`OS_LOWEST_PRIO`</mark><mark style="color:blue;">, inclusive.</mark> Please note again that, `OS_LOWEST_PRIO` is reserved by µCOS-II’s idle task. Your application will not be able to call `OSTaskCreateExt()` and create a task at priority `OS_LOWEST_PRIO` because it would have already been ‘reserved’ for the idle task by `OSInit()`. In this case, `OSTaskCreateExt()` would return `OS_PRIO_EXIST`.

(2) Next, `OSTaskCreateExt()` makes sure that a task has not already been created at the desired priority. With µCOS-II, all tasks must have a unique priority.

(3) If the desired priority is free, then µCOS-II reserves the priority by placing a non-NULL pointer in `OSTCBPrioTbl[]`.

(4) This allows `OSTaskCreateExt()` to re-enable interrupts while it sets up the rest of the data structures for the task.

(5) `OSTaskCreateExt()` then calls `OSTaskStkInit()`, which is responsible for setting up the task stack. This function is processor specific and is found in `OS_CPU_C.C`. Refer to Chapter 13, Porting µCOS-II, for details on how to implement `OSTaskStkInit()`. If you already have a port of µCOS-II for the processor you are intending to use, then you don’t need to be concerned about implementation details. `OSTaskStkInit()` returns the new top-of-stack (psp) which will be saved in the task’s `OS_TCB`. µCOS-II supports processors that have stacks that grow either from high to low memory or from low to high memory (see section 4.02, Task Stacks). When you call `OSTaskCreateExt()`, you must know how the stack grows (see `OS_CPU.H` of the processor you are using) because you must pass the task’s top-of-stack, which can either be the lowest memory location of the stack (when `OS_STK_GROWTH` is 0) or the highest memory location of the stack (when `OS_STK_GROWTH` is 1), to `OSTaskCreateExt()`.

(6) Once `OSTaskStkInit()` has completed setting up the stack, `OSTaskCreateExt()` calls `OS_TCBInit()` to obtain and initialize an `OS_TCB` from the pool of free `OS_TCBs`. The code for `OS_TCBInit()` is described in section 3.03, Task Control Blocks.

(10) If `OS_TCBInit()` failed, the priority level is relinquished by setting the entry in `OSTCBPrioTbl[prio]` to 0.

(7) & (8) & (9) Finally, if `OSTaskCreateExt()` is called once multitasking has started (i.e., `OSRunning` is set to TRUE), the scheduler is called to determine whether the created task has a higher priority than its creator. Creating a higher priority task results in a context switch to the new task. If the task was created before multitasking started \[i.e., you did not call `OSStart()` yet], the scheduler is not called.

### Task Stacks <a href="#taskmanagement-taskstacks" id="taskmanagement-taskstacks"></a>

Each task must have its own stack space. A stack must be declared as being of type `OS_STK` and must consist of contiguous memory locations. You can allocate stack space either statically (at compile time) or dynamically (at run time). A static stack declaration is shown in Listings 4.3 and 4.4. Either declaration is made outside a function

{% code title="Listing 4.3 Static stack" %}
```c
static OS_STK  MyTaskStack[stack_size];
```
{% endcode %}

or

{% code title="Listing 4.4 Static stack" %}
```c
OS_STK  MyTaskStack[stack_size];
```
{% endcode %}

You can allocate stack space dynamically by using the C compiler’s malloc() function as shown in Listing 4.5. However, you must be careful with fragmentation. Specifically, if you create and delete tasks, your memory allocator may not be able to return a stack for your task(s) because the heap eventually becomes fragmented.

{% code title="Listing 4.5 Using malloc() to allocate stack space for a task" fullWidth="false" %}
```c
OS_STK  *pstk;
 
 
pstk = (OS_STK *)malloc(stack_size);
if (pstk != (OS_STK *)0) {      /* Make sure malloc() has enough space */
    Create the task;
}
```
{% endcode %}

<figure><img src="https://micrium.atlassian.net/wiki/download/thumbnails/163900/CH04-Fig01-Fragmentation.png?version=1&#x26;modificationDate=1428669236000&#x26;cacheVersion=1&#x26;api=v2&#x26;width=300&#x26;height=230" alt=""><figcaption><p>Figure 4.1 Fragmentation</p></figcaption></figure>

(1) Figure 4.1 illustrates a heap containing 3Kb of available memory that can be allocated with malloc(). For the sake of discussion, you create three tasks (tasks A, B, and C), each requiring 1Kb.

(2) Assume that the first 1Kb is given to task A, the second to task B, and the third to task C.

(3) Your application then deletes task A and task C and relinquishes the memory to the heap using free(). Your heap now has 2Kb of memory free, but it’s not contiguous. This means that you cannot create another task (i.e., task D) that requires 2 Kb because your heap is fragmented. If, however, you never delete a task, the use of malloc() is perfectly acceptable.

Because µCOS-II supports processors with stacks that grow either from high to low memory or from low to high memory, you must know how the stack grows when you call either `OSTaskCreate()` or `OSTaskCreateExt()` because you need to pass the task’s top-of-stack to these functions. When `OS_STK_GROWTH` is set to 0 in `OS_CPU.H` , you need to pass the lowest memory location of the stack to the task create function as shown in Listing 4.6.

{% code title="Listing 4.6 Stack grows from low to high memory" %}
```c
OS_STK  TaskStk[TASK_STK_SIZE];
 
OSTaskCreate(task, pdata, &TaskStk[0], prio);
```
{% endcode %}

When `OS_STK_GROWTH` is set to 1 in `OS_CPU.H`, you need to pass the highest memory location of the stack to the task create function as shown in Listing 4.7.

{% code title="Listing 4.7 Stack grows from high to low memory" %}
```c
OS_STK  TaskStk[TASK_STK_SIZE];
 
OSTaskCreate(task, pdata, &TaskStk[TASK_STK_SIZE-1], prio);
```
{% endcode %}

This requirement affects code portability(可移植性). If you need to port your code from a processor architecture that supports a downward-growing stack to one that supports an upward-growing stack, you may need to make your code handle both cases. Specifically, Listings 4.6 and 4.7 are rewritten as shown in Listing 4.8.

{% code title="Listing 4.8 Supporting stacks that grow in either direction" %}
```c
OS_STK  TaskStk[TASK_STK_SIZE];
 
 
#if OS_STK_GROWTH == 0
    OSTaskCreate(task, pdata, &TaskStk[0], prio);
#else
    OSTaskCreate(task, pdata, &TaskStk[TASK_STK_SIZE-1], prio);
#endif
```
{% endcode %}

The size of the stack needed by your task is application specific. When sizing the stack, however, you must account for nesting(嵌套) of all the functions called by your task, the number of local variables that will be allocated by all functions called by your task, and the stack requirements for all nested interrupt service routines. In addition, your stack must be able to store all CPU registers.

### Stack Checking, OSTaskStkChk() <a href="#taskmanagement-stackchecking-ostaskstkchk" id="taskmanagement-stackchecking-ostaskstkchk"></a>

Sometimes it is necessary to determine how much stack space a task actually uses. This allows you to reduce the amount of RAM needed by your application code by not overallocating stack space. µCOS-II provides `OSTaskStkChk()`, which provides you with this valuable information.

In order to use the µCOS-II stack-checking facilities, you must do the following.

* Set `OS_TASK_CREATE_EXT` to 1 in `OS_CFG.H`.
* Create a task using `OSTaskCreateExt()` and give the task much more space than you think it really needs. You can call `OSTaskStkChk()` for any task, from any task.
* Set the opt argument in `OSTaskCreateExt()` to `OS_TASK_OPT_STK_CLR + OS_TASK_OPT_STK_CLR`. Note that if your startup code clears all RAM and you never delete tasks once they are created, you don’t need to set the `OS_TASK_OPT_STK_CLR` option. This reduces the execution time of `OSTaskCreateExt()`.
* Call `OSTaskStkChk()` from a task by specifying the priority of the task you want to check. You can inquire about any task stack not just the running task.

<figure><img src="https://micrium.atlassian.net/wiki/download/thumbnails/163900/Figure%204.2.png?version=2&#x26;modificationDate=1508769091000&#x26;cacheVersion=1&#x26;api=v2&#x26;width=500&#x26;height=391" alt=""><figcaption><p>Figure 4.2 Stack checking</p></figcaption></figure>

(1) In Figure 4.2, I assume that the stack grows from high memory to low memory (i.e., `OS_STK_GROWTH` is set to 1) but the following discussion applies equally well to a stack growing in the opposite direction. µCOS-II determines stack growth by looking at the contents of the stack itself. Stack checking is performed on demand as opposed to continuously.

(2) To perform stack checking, µCOS-II requires that the stack be filled with zeros when the task is created.

(3) & (4) Also, µCOS-II needs to know the location of the bottom-of-stack (BOS) and the size of the stack you assigned to the task. These two values are stored in the task’s `OS_TCB` when the task is created, but only if created with `OSTaskCreateExt()` .

(5) `OSTaskStkChk()` computes the amount of free stack space by “walking” from the bottom of the stack and counting the number of zero-value entries on the stack until a nonzero value is found. Note that stack entries are checked using the data type of the stack (see `OS_STK` in `OS_CPU.H`). In other words, if a stack entry is 32 bits wide, the comparison for a zero value is done using 32 bits.

(6) & (8) The amount of stack space used is obtained by subtracting the number of zero-value entries from the stack size you specified in `OSTaskCreateExt()` . `OSTaskStkChk()` actually places the number of bytes free and the number of bytes used in a data structure of type `OS_STK_DATA` (see `uCOS_II.H` ).

(7) Note that at any given time, the stack pointer for the task being checked may be pointing somewhere between the initial top-of-stack (TOS) and the deepest stack growth.

(5) Also, every time you call `OSTaskStkChk()`, you may get a different value for the amount of free space on the stack until your task has reached its deepest growth.

You need to run the application long enough and under your worst case conditions to get proper numbers. Once `OSTaskStkChk()` provides you with the worst case stack requirement, you can go back and set the final size of your stack. You should accommodate system expansion, so make sure you allocate between 10 and 100 percent more stack than what `OSTaskStkChk()` reports. What you should get from stack checking is a ballpark figure(大概的数字); you are not looking for an exact stack usage.

The code for `OSTaskStkChk()` is shown in Listing 4.9. The data structure `OS_STK_DATA` (see `uCOS_II.H`) is used to hold information about the task stack. I decided to use a data structure for two reasons. First, I consider `OSTaskStkChk()` to be a query-type function, and I wanted to have all query functions work the same way — return data about the query in a data structure. Second, passing data in a data structure is efficient and allows me to add additional fields in the future without changing the API (Application Programming Interface) of `OSTaskStkChk()`. For now, `OS_STK_DATA` only contains two fields: `OSFree` and `OSUsed`. As you can see, you invoke `OSTaskStkChk()` by specifying the priority of the task you want to perform stack checking on.

{% code title="Listing 4.9 Stack-checking function" %}
```c
INT8U  OSTaskStkChk (INT8U prio, OS_STK_DATA *pdata)
{
#if OS_CRITICAL_METHOD == 3                      
    OS_CPU_SR  cpu_sr;
#endif
    OS_TCB    *ptcb;
    OS_STK    *pchk;
    INT32U     free;
    INT32U     size;
 
 
#if OS_ARG_CHK_EN > 0u
    if (prio > OS_LOWEST_PRIO) {                                (1)
        if (prio != OS_PRIO_SELF) {
            return (OS_ERR_PRIO_INVALID);
        }
    }
    if (p_stk_data == (OS_STK_DATA *)0) {              
        return (OS_ERR_PDATA_NULL);
    }
#endif
    pdata->OSFree = 0u;                                          
    pdata->OSUsed = 0u;
    OS_ENTER_CRITICAL();
    if (prio == OS_PRIO_SELF) {                                 (2)
        prio = OSTCBCur->OSTCBPrio;
    }
    ptcb = OSTCBPrioTbl[prio];
    if (ptcb == (OS_TCB *)0) {                                  (3)
        OS_EXIT_CRITICAL();
        return (OS_ERR_TASK_NOT_EXIST);
    }
    if (ptcb == OS_TCB_RESERVED) {
        OS_EXIT_CRITICAL();
        return (OS_ERR_TASK_NOT_EXIST);
    }

    if ((ptcb->OSTCBOpt & OS_TASK_OPT_STK_CHK) == 0u) {         (4)
        OS_EXIT_CRITICAL();
        return (OS_ERR_TASK_OPT);
    }
    free = 0u;                                                  (5)
    size = ptcb->OSTCBStkSize;
    pchk = ptcb->OSTCBStkBottom;
    OS_EXIT_CRITICAL();
#if OS_STK_GROWTH == 1u
    while (*pchk++ == (OS_STK)0) {                    
        free++;
    }
#else
    while (*pchk-- == (OS_STK)0) {
        free++;
    }
#endif
    pdata->OSFree = free * sizeof(OS_STK);                      (6)
    pdata->OSUsed = (size - free) * sizeof(OS_STK);   
    return (OS_ERR_NONE);
}
```
{% endcode %}

(1) If `OS_ARG_CHK_EN` is set to 1 in `OS_CFG.H`, `OSTaskStkChk()` verifies that the priority is within valid range.

(2) If you specify `OS_PRIO_SELF`, it is assumed that you want to know the stack information about the current task.

(3) Obviously, the task must exist. Simply checking for the presence of a non-NULL pointer in `OSTCBPrioTbl[]` ensures this.

(4) <mark style="color:blue;">To perform stack checking, you must have created the task using</mark> <mark style="color:blue;"></mark><mark style="color:blue;">`OSTaskCreateExt()`</mark> <mark style="color:blue;"></mark><mark style="color:blue;">and you must have passed the option</mark> <mark style="color:blue;"></mark><mark style="color:blue;">`OS_TASK_OPT_STK_CHK`</mark><mark style="color:blue;">.</mark> If you would called `OSTaskStkChk()` from a task that was created by `OSTaskCreate()` (instead of `OSTaskCreateExt()`) then the opt argument (passed to `OS_TCBInit()`) would have been 0 and the test would fail.

(5) If all the proper conditions are met, `OSTaskStkChk()` computes the free stack space as described above by walking from the bottom of stack until a nonzero stack entry is encountered.

(6) Finally, the information that is stored in `OS_STK_DATA` is computed. Note that the function computes the actual number of bytes free and the number of bytes used on the stack as opposed to the number of elements. Obviously, the actual stack size (in bytes) can be obtained by adding these two values.

### Deleting a Task, OSTaskDel() <a href="#taskmanagement-deletingatask-ostaskdel" id="taskmanagement-deletingatask-ostaskdel"></a>

Sometimes it is necessary to delete a task. Deleting a task means that the task will be returned to the DORMANT state (see section 3.02, Task States) and does not mean that the code for the task will be deleted. The task code is simply no longer scheduled by µCOS-II. You delete a task by calling `OSTaskDel()` (Listing 4.10).

{% code title="Listing 4.10 Task delete" %}
```c
INT8U  OSTaskDel (INT8U prio)
{
#if OS_CRITICAL_METHOD == 3                      
    OS_CPU_SR     cpu_sr;
#endif
 
#if OS_EVENT_EN > 0
    OS_EVENT     *pevent;
#endif    
#if (OS_FLAG_EN > 0u) && (OS_MAX_FLAGS > 0u)
    OS_FLAG_NODE *pnode;
#endif
    OS_TCB       *ptcb;
 
 
    if (OSIntNesting > 0) {                                                (1)
        return (OS_ERR_TASK_DEL_ISR);
    }
    if (prio == OS_TASK_IDLE_PRIO) {                                       (2)                               
        return (OS_ERR_TASK_DEL_IDLE);
    }
#if OS_ARG_CHK_EN > 0
    if (prio >= OS_LOWEST_PRIO && prio != OS_PRIO_SELF) {                  (3)
        return (OS_ERR_PRIO_INVALID);
    }
#endif
    OS_ENTER_CRITICAL();
    if (prio == OS_PRIO_SELF) {                                            (4)
        prio = OSTCBCur->OSTCBPrio;                             
    }
    ptcb = OSTCBPrioTbl[prio];
    if (ptcb != (OS_TCB *)0) {                                             (5)
        OS_EXIT_CRITICAL();
        return (OS_ERR_TASK_NOT_EXIST);
    }
    if (ptcb == OS_TCB_RESERVED) {                      
        OS_EXIT_CRITICAL();
        return (OS_ERR_TASK_DEL);
    }

    OSRdyTbl[ptcb->OSTCBY] &= (OS_PRIO)~ptcb->OSTCBBitX;                   (6)
    if (OSRdyTbl[ptcb->OSTCBY] == 0u) {                
        OSRdyGrp           &= (OS_PRIO)~ptcb->OSTCBBitY;
    }

#if (OS_EVENT_EN)
    if (ptcb->OSTCBEventPtr != (OS_EVENT *)0) {                            (7)
        OS_EventTaskRemove(ptcb, ptcb->OSTCBEventPtr);  /
    }
#if (OS_EVENT_MULTI_EN > 0u)
    if (ptcb->OSTCBEventMultiPtr != (OS_EVENT **)0) {   
        OS_EventTaskRemoveMulti(ptcb, ptcb->OSTCBEventMultiPtr);
    }
#endif
#endif

#if (OS_FLAG_EN > 0u) && (OS_MAX_FLAGS > 0u)
    pnode = ptcb->OSTCBFlagNode;                                           (8)
    if (pnode != (OS_FLAG_NODE *)0) {                       
        OS_FlagUnlink(pnode);                               
    }
#endif
    ptcb->OSTCBDly      = 0u;                                              (9)
    ptcb->OSTCBStat     = OS_STAT_RDY;                                    (10)
    ptcb->OSTCBStatPend = OS_STAT_PEND_OK;

    if (OSLockNesting < 255u) {                                           (11)
        OSLockNesting++;
    }
    OS_EXIT_CRITICAL();                                                   (12)
    OS_Dummy();                                                           (13)
    OS_ENTER_CRITICAL();                                    
    if (OSLockNesting > 0u) {                                             (14)
        OSLockNesting--;
    }
    OSTaskDelHook(ptcb);                                                  (15)

#if OS_TASK_CREATE_EXT_EN > 0u
#if defined(OS_TLS_TBL_SIZE) && (OS_TLS_TBL_SIZE > 0u)
    OS_TLS_TaskDel(ptcb);                               
#endif
#endif

    OSTaskCtr--;                                                          (16)
    OSTCBPrioTbl[prio] = (OS_TCB *)0;                                     (17)
    if (ptcb->OSTCBPrev == (OS_TCB *)0) {                                 (18)
        ptcb->OSTCBNext->OSTCBPrev = (OS_TCB *)0;
        OSTCBList                  = ptcb->OSTCBNext;
    } else {
        ptcb->OSTCBPrev->OSTCBNext = ptcb->OSTCBNext;
        ptcb->OSTCBNext->OSTCBPrev = ptcb->OSTCBPrev;
    }
    ptcb->OSTCBNext     = OSTCBFreeList;                                  (19)
    OSTCBFreeList       = ptcb;
#if OS_TASK_NAME_EN > 0u
    ptcb->OSTCBTaskName = (INT8U *)(void *)"?";
#endif

    OS_EXIT_CRITICAL();
    if (OSRunning == OS_TRUE) {
        OS_Sched();                                                       (20)                                 
    }
    return (OS_ERR_NONE);
}
```
{% endcode %}

(1) `OSTaskDel()` starts off by making sure you are not attempting to delete a task from within an ISR because that’s not allowed.

(2) `OSTaskDel()` checks that you are not attempting to delete the idle task because this is also not allowed.

(3) You are allowed to delete the statistic task (`OS_LOWEST_PRIO-1`) and all higher priority tasks (i.e. the task priority has a lower number).

(4) The caller can delete itself by specifying `OS_PRIO_SELF` as the argument.

(5) `OSTaskDel()` verifies that the task to delete does in fact exist . This test obviously will pass if you specified `OS_PRIO_SELF`. I didn’t want to create a separate case for this situation because it would have increased code size and thus execution time. If `OS_PRIO_SELF` is specified, we simply obtain the priority of the current task which is stored in its OS\_TCB.

Once all conditions are satisfied, the `OS_TCB` is removed from all possible µCOS-II data structures. `OSTaskDel()` does this in two parts to reduce interrupt latency(中断延迟).

(6) First, if the task is in the ready list, it is removed.

(7) If the task is in a list waiting for a mutex, mailbox, queue, or semaphore, it is removed from that list.

(8) If the task is in a list waiting for an event flag, it is removed from that list.

(9) Next, `OSTaskDel()` forces the delay count to zero to make sure that the tick ISR will not ready this task once you re-enable interrupts.

(10) `OSTaskDel()` sets the task’s `.OSTCBStat` flag to `OS_STAT_RDY`. Note that `OSTaskDel()` is not trying to make the task ready, it is simply preventing another task or an ISR from resuming this task \[i.e., in case the other task or ISR calls `OSTaskResume()`]. This situation could occur because `OSTaskDel()` will be re-enabling interrupts (see L4.10(12)), so an ISR can make a higher priority task ready, which could resume the task you are trying to delete. Instead of setting the task’s `.OSTCBStat` flag to `OS_STAT_RDY`, I simply could have cleared the `OS_STAT_SUSPEND` bit (which would have been clearer), but this takes slightly more processing time.

(11) At this point, the task to delete cannot be made ready to run by another task or an ISR because it’s been removed from the ready list, it’s not waiting for an event to occur, it’s not waiting for time to expire, and it cannot be resumed. For all intents and purposes, the task is DORMANT. Because of this, `OSTaskDel()` must prevent the scheduler from switching to another task because if the current task is almost deleted, it could not be rescheduled!

(12) At this point, `OSTaskDel()` re-enables interrupts in order to reduce interrupt latency. `OSTaskDel()` could thus service an interrupt, but because it incremented `OSLockNesting`, the ISR would return to the interrupted task. Note that `OSTaskDel()` is still not done with the deletion process because it needs to unlink the `OS_TCB` from the TCB chain and return the `OS_TCB` to the free `OS_TCB` list.

(13) Note also that I call the dummy function `OS_Dummy()` immediately after calling `OS_EXIT_CRITICAL()`. I do this because I want to make sure that the processor executes at least one instruction with interrupts enabled. On many processors, executing an interrupt enable instruction forces the CPU to have interrupts disabled until the end of the next instruction! The Intel 80x86 and Zilog Z-80 processors actually work like this. Enabling and immediately disabling interrupts would behave just as if I didn’t enable interrupts. This would of course increase interrupt latency. Calling `OS_Dummy()` thus ensures that I execute a call and a return instruction before re-disabling interrupts. You could certainly replace `OS_Dummy()` with a macro that executes a “no-operation” instruction and thus slightly reduce the execution time of `OSTaskDel()`. I didn’t think it was worth the effort of creating yet another macro that would require porting.

(14) `OSTaskDel()` can now continue with the deletion process of the task. After `OSTaskDel()` re-disables interrupts, `OSTaskDel()` re-enables scheduling by decrementing the lock nesting counter.

(15) `OSTaskDel()` then calls the user-definable task delete hook `OSTaskDelHook()`. This allows user-defined `OS_TCB` extensions to be relinquished.

(16) Next, `OSTaskDel()` decrements the task counter to indicate that there is one less task being managed by µCOS-II.

(17) `OSTaskDel()` removes the `OS_TCB` from the priority table by simply replacing the link to the `OS_TCB` of the task being deleted with a NULL pointer.

(18) `OSTaskDel()` then removes the `OS_TCB` of the task being deleted from the doubly linked list of `OS_TCBs` that starts at `OSTCBList`. Note that there is no need to check for the case where `ptcb->OSTCBNext` == 0 because `OSTaskDel()` cannot delete the idle task, which always happens to be at the end of the chain.

(19) The `OS_TCB` is returned to the free list of `OS_TCBs` to allow another task to be created.

(20) Last, but not least, the scheduler is called to see if a higher priority task has been made ready to run by an ISR that would have occurred when `OSTaskDel()` re-enabled interrupts at step \[L4.11(12)].

### Requesting to Delete a Task, OSTaskDelReq() <a href="#taskmanagement-requestingtodeleteatask-ostaskdelreq" id="taskmanagement-requestingtodeleteatask-ostaskdelreq"></a>

Sometimes, a task owns resources such as memory buffers or a semaphore. If another task attempts to delete this task, the resources are not freed and thus are lost. This would lead to memory leaks which is not acceptable for just about any embedded system. In this type of situation, you somehow need to tell the task that owns these resources to delete itself when it’s done with the resources. You can accomplish this with the `OSTaskDelReq()` function. Both the requestor and the task to be deleted need to call `OSTaskDelReq()`. The requestor code is shown in Listing 4.11.

{% code title="Listing 4.11 Requester code requesting a task to delete itself" %}
```c
void RequestorTask (void *pdata)
{
    INT8U err;
 
 
    pdata = pdata;
    for (;;) {
        /* Application code */
        if ('TaskToBeDeleted()' needs to be deleted) {                     (1)
            while (OSTaskDelReq(TASK_TO_DEL_PRIO) != OS_TASK_NOT_EXIST) {  (2)
                OSTimeDly(1);                                              (3)
            }
        }
        /* Application code */                                             (4)
    }
}
```
{% endcode %}

(1) The task that makes the request needs to determine what conditions would cause a request for the task to be deleted. In other words, your application determines what conditions lead to this decision.

(2) If the task needs to be deleted, call `OSTaskDelReq()` by passing the priority of the task to be deleted. If the task to delete does not exist, `OSTaskDelReq()` returns `OS_TASK_NOT_EXIST`. You would get this if the task to delete has already been deleted or has not been created yet. If the return value is `OS_NO_ERR`, the request has been accepted but the task has not been deleted yet. You may want to wait until the task to be deleted does in fact delete itself.

(3) You can do this by delaying the requestor for a certain amount of time, as I did in. I decided to delay for one tick, but you can certainly wait longer if needed.

(4) When the requested task eventually deletes itself, the return value in L4.11(2) is `OS_TASK_NOT_EXIST` and the loop exits.

The pseudocode(伪代码) for the task that needs to delete itself is shown in Listing 4.12. This task basically polls a flag that resides inside the task’s `OS_TCB`. The value of this flag is obtained by calling `OSTaskDelReq(OS_PRIO_SELF)`.

{% code title="Listing 4.12 Task requesting to delete itself" %}
```c
void TaskToBeDeleted (void *pdata)
{
    INT8U err;
 
 
    pdata = pdata;
    for (;;) {
        /* Application code */
        if (OSTaskDelReq(OS_PRIO_SELF) == OS_TASK_DEL_REQ) {                  (1)
            Release any owned resources;                                      (2)
            De-allocate any dynamic memory;
            OSTaskDel(OS_PRIO_SELF);                                          (3)
        } else {
            /* Application code */
        }
    }
}
```
{% endcode %}

(1) When `OSTaskDelReq()` returns `OS_TASK_DEL_REQ` to its caller, it indicates that another task has requested that this task needs to be deleted.

(2) & (3) In this case, the task to be deleted releases any resources owned and calls `OSTaskDel(OS_PRIO_SELF)` to delete itself. As previously mentioned, the code for the task is not actually deleted. Instead, µC/OS-II simply does not schedule the task for execution. In other words, the task code will no longer run. You can, however, recreate the task by calling either `OSTaskCreate()` or `OSTaskCreateExt()`.

The code for `OSTaskDelReq()` is shown in Listing 4.13. As usual, `OSTaskDelReq()` needs to check for boundary conditions.

```
INT8U  OSTaskDelReq (INT8U prio)
{
#if OS_CRITICAL_METHOD == 3                      
    OS_CPU_SR  cpu_sr;
#endif
    BOOLEAN    stat;
    INT8U      err;
    OS_TCB    *ptcb;
 
 
    if (prio == OS_IDLE_PRIO) {                                    (1)
        return (OS_ERR_TASK_DEL_IDLE);
    }

#if OS_ARG_CHK_EN > 0
    if (prio >= OS_LOWEST_PRIO && prio != OS_PRIO_SELF) {          (2)
        return (OS_ERR_PRIO_INVALID);
    }
#endif
    if (prio == OS_PRIO_SELF) {                                    (3)
        OS_ENTER_CRITICAL();                                  
        stat = OSTCBCur->OSTCBDelReq;                         
        OS_EXIT_CRITICAL();
        return (stat);
    }
    OS_ENTER_CRITICAL();
    ptcb = OSTCBPrioTbl[prio];

    if (ptcb == (OS_TCB *)0) {                                     (4)                                 
        OS_EXIT_CRITICAL();
        return (OS_ERR_TASK_NOT_EXIST);                            (6)
    }
    if (ptcb == OS_TCB_RESERVED) {                              
        OS_EXIT_CRITICAL();
        return (OS_ERR_TASK_DEL);
    }
    ptcb->OSTCBDelReq = OS_ERR_TASK_DEL_REQ;                       (5)
    OS_EXIT_CRITICAL();
    return (OS_ERR_NONE);
}
```

(1) First, `OSTaskDelReq()` notifies the caller in case he requests to delete the idle task.

(2) Next, it must ensure that the caller is not trying to request to delete an invalid priority.

(3) If the caller is the task to be deleted, the flag stored in the `OS_TCB` is returned.

(4) & (5) If you specified a task with a priority other than `OS_PRIO_SELF` and the task exists, `OSTaskDelReq()` sets the internal flag for that task.

(6) If the task does not exist, `OSTaskDelReq()` returns `OS_TASK_NOT_EXIST` to indicate that the task must have deleted itself.

### Changing a Task’s Priority, OSTaskChangePrio() <a href="#taskmanagement-changingataskspriority-ostaskchangeprio" id="taskmanagement-changingataskspriority-ostaskchangeprio"></a>

When you create a task, you assign the task a priority. At run time, you can change the priority of any task by calling `OSTaskChangePrio()`. In other words, µC/OS-II allows you to change priorities dynamically. The code for `OSTaskChangePrio()` is shown in Listing 4.14.

```
INT8U  OSTaskChangePrio (INT8U oldprio, INT8U newprio)
{
#if OS_CRITICAL_METHOD == 3                      
    OS_CPU_SR    cpu_sr;
#endif
 
#if OS_EVENT_EN > 0
    OS_EVENT    *pevent;
#endif
 
    OS_TCB      *ptcb;
    INT8U        x;
    INT8U        y;
    INT8U        bitx;
    INT8U        bity;
 
 
 
#if OS_ARG_CHK_EN > 0u
    if (oldprio >= OS_LOWEST_PRIO) {                                          (1)
        if (oldprio != OS_PRIO_SELF) {
            return (OS_ERR_PRIO_INVALID);
        }
    }
    if (newprio >= OS_LOWEST_PRIO) {
        return (OS_ERR_PRIO_INVALID);
    }
#endif

    OS_ENTER_CRITICAL();
    if (OSTCBPrioTbl[newprio] != (OS_TCB *)0) {                               (2)
        OS_EXIT_CRITICAL();
        return (OS_ERR_PRIO_EXIST);
    } 
    if (oldprio == OS_PRIO_SELF) {                                           
        oldprio = OSTCBCur->OSTCBPrio;                      
    }

    ptcb = OSTCBPrioTbl[oldprio];                                             (3)
    if (ptcb == (OS_TCB *)0) {                              
        OS_EXIT_CRITICAL();                                 
        return (OS_ERR_PRIO);
    }
    if (ptcb == OS_TCB_RESERVED) {                          
        OS_EXIT_CRITICAL();                                 
        return (OS_ERR_TASK_NOT_EXIST);
    }
#if OS_LOWEST_PRIO <= 63u
    y_new                 = (INT8U)(newprio >> 3u);                           (4)   
    x_new                 = (INT8U)(newprio & 0x07u);
#else
    y_new                 = (INT8U)((INT8U)(newprio >> 4u) & 0x0Fu);
    x_new                 = (INT8U)(newprio & 0x0Fu);
#endif
    bity_new              = (OS_PRIO)(1uL << y_new);
    bitx_new              = (OS_PRIO)(1uL << x_new);
    OSTCBPrioTbl[oldprio] = (OS_TCB *)0;                    
    OSTCBPrioTbl[newprio] =  ptcb;                          
    y_old                 =  ptcb->OSTCBY;
    bity_old              =  ptcb->OSTCBBitY;
    bitx_old              =  ptcb->OSTCBBitX;
    if ((OSRdyTbl[y_old] &   bitx_old) != 0u) {                               (5)       
         OSRdyTbl[y_old] &= (OS_PRIO)~bitx_old;
         if (OSRdyTbl[y_old] == 0u) {
             OSRdyGrp &= (OS_PRIO)~bity_old;
         }
         OSRdyGrp        |= bity_new;                       
         OSRdyTbl[y_new] |= bitx_new;
    }
#if (OS_EVENT_EN)
    pevent = ptcb->OSTCBEventPtr;                                             (6)
    if (pevent != (OS_EVENT *)0) {
        pevent->OSEventTbl[y_old] &= (OS_PRIO)~bitx_old;    
        if (pevent->OSEventTbl[y_old] == 0u) {
            pevent->OSEventGrp    &= (OS_PRIO)~bity_old;
        }
        pevent->OSEventGrp        |= bity_new;              
        pevent->OSEventTbl[y_new] |= bitx_new;
    }
#if (OS_EVENT_MULTI_EN > 0u)
    if (ptcb->OSTCBEventMultiPtr != (OS_EVENT **)0) {                         
        pevents =  ptcb->OSTCBEventMultiPtr;
        pevent  = *pevents;
        while (pevent != (OS_EVENT *)0) {
            pevent->OSEventTbl[y_old] &= (OS_PRIO)~bitx_old;   
            if (pevent->OSEventTbl[y_old] == 0u) {
                pevent->OSEventGrp    &= (OS_PRIO)~bity_old;
            }
            pevent->OSEventGrp        |= bity_new;          
            pevent->OSEventTbl[y_new] |= bitx_new;
            pevents++;
            pevent                     = *pevents;
        }
    }
#endif
#endif
    ptcb->OSTCBPrio = newprio;                                                (7)                          
    ptcb->OSTCBY    = y_new;
    ptcb->OSTCBX    = x_new;
    ptcb->OSTCBBitY = bity_new;
    ptcb->OSTCBBitX = bitx_new;
    OS_EXIT_CRITICAL();
    if (OSRunning == OS_TRUE) {
        OS_Sched();                                         
    }
    return (OS_ERR_NONE);
}
```

(1) You cannot change the priority of the idle task. You can change either the priority of the calling task or another task. To change the priority of the calling task, either specify the old priority of that task or specify `OS_PRIO_SELF`, and `OSTaskChangePrio()` will determine what the priority of the calling task is for you. You must also specify the new (i.e., desired) priority.

(2) Because µC/OS-II cannot have multiple tasks running at the same priority, `OSTaskChangePrio()` needs to check that the new desired priority is available.

(3) Here we are making sure that the priority we are changing does indeed exist.

(4) `OSTaskChangePrio()` precomputes some values that are stored in the task’s `OS_TCB`. These values are used to put or remove the task in or from the ready list (see section 3.04, Ready List).

(5) If the task that we are changing for is ready to run then we need to remove the task from the ready list at the current priority and insert it in the ready list at the new priority.

(6) If the task is not ready, it could be waiting on a semaphore, mailbox, or queue. `OSTaskChangePrio()` knows that the task is waiting for one of these events if the `OSTCBEventPtr` is non-NULL. If the task is waiting for an event, `OSTaskChangePrio()` must remove the task from the wait list (at the old priority) of the event control block (see Chapter 6, Event Control Blocks) and insert the task back into the wait list, but this time at the new priority. The task could be waiting for time to expire (see Chapter 5, Time Management) or the task could be suspended \[see section 4.07, Suspending a Task, `OSTaskSuspend()`].

(7) Pre-computed are then saved in the task's TCB.

After `OSTaskChangePrio()` exits the critical section, the scheduler is called in case the new priority is higher than the old priority or the priority of the calling task.

### Suspending a Task, OSTaskSuspend() <a href="#taskmanagement-suspendingatask-ostasksuspend" id="taskmanagement-suspendingatask-ostasksuspend"></a>

Sometimes it is useful to explicitly suspend the execution of a task. This is accomplished with the `OSTaskSuspend()` function call. A suspended task can only be resumed by calling the `OSTaskResume()` function call. Task suspension is additive. This means that if the task being suspended is also waiting for time to expire, the suspension needs to be removed and the time needs to expire in order for the task to be ready to run. A task can suspend either itself or another task.

The code for `OSTaskSuspend()` is shown in Listing 4.15.

```
INT8U  OSTaskSuspend (INT8U prio)
{
#if OS_CRITICAL_METHOD == 3                      
    OS_CPU_SR  cpu_sr;
#endif
    BOOLEAN    self;
    OS_TCB    *ptcb;
 
 
#if OS_ARG_CHK_EN > 0
    if (prio == OS_IDLE_PRIO) {                                   (1)
        return (OS_ERR_TASK_SUSPEND_IDLE);
    }
    if (prio >= OS_LOWEST_PRIO && prio != OS_PRIO_SELF) {         (2)
        return (OS_ERR_PRIO_INVALID);
    }
#endif
    OS_ENTER_CRITICAL();
    if (prio == OS_PRIO_SELF) {                                   (3)
        prio = OSTCBCur->OSTCBPrio;
        self = OS_TRUE;
    } else if (prio == OSTCBCur->OSTCBPrio) {                     (4)
        self = OS_TRUE;
    } else {
        self = OS_FALSE;                                           
    }
    ptcb = OSTCBPrioTbl[prio];
    if (ptcb == (OS_TCB *)0) {                                    (5)
        OS_EXIT_CRITICAL();
        return (OS_ERR_TASK_SUSPEND_PRIO);
    }

    OSRdyTbl[y] &= (OS_PRIO)~ptcb->OSTCBBitX;                     (6)
    if (OSRdyTbl[y] == 0u) {
        OSRdyGrp &= (OS_PRIO)~ptcb->OSTCBBitY;
    }
    ptcb->OSTCBStat |= OS_STAT_SUSPEND;                           (7)
    OS_EXIT_CRITICAL();
    if (self == OS_TRUE) {                                         
        OS_Sched();                                               (8)
    }
    return (OS_ERR_NONE);
}
```

(1) `OSTaskSuspend()` ensures that your application is not attempting to suspend the idle task.

(2) Next, you must specify a valid priority. Remember that the highest valid priority number (i.e., lowest priority) is `OS_LOWEST_PRIO`. Note that you can suspend the statistic task. You may have noticed that the first test \[L4.15(1)] is replicated in \[L4.15(2)]. I did this to be backward compatible with µC/OS. The first test could be removed to save a little bit of processing time, but this is really insignificant so I decided to leave it.

(3) Next, `OSTaskSuspend()` checks to see if you specified to suspend the calling task by specifying `OS_PRIO_SELF`. In this case, the current task’s priority is retrieved from its `OS_TCB`.

(4) You could also decided to suspend the calling task by specifying its priority. In both of these cases, the scheduler needs to be called. This is why I created the local variable self, which will be examined at the appropriate time. If you are not suspending the calling task, then `OSTaskSuspend()` does not need to run the scheduler because the calling task is suspending a lower priority task.

(5) `OSTaskSuspend()` then checks to see that the task to suspend exists.

(6) If so, it is removed from the ready list. Note that the task to suspend may not be in the ready list because it could be waiting for an event or for time to expire. In this case, the corresponding bit for the task to suspend in `OSRdyTbl[]` would already be cleared (i.e., 0). Clearing it again is faster than checking to see if it’s clear and then clearing it if it’s not.

(7) Now `OSTaskSuspend()` sets the `OS_STAT_SUSPEND` flag in the task’s `OS_TCB` to indicate that the task is now suspended.

(8) Finally, `OSTaskSuspend()` calls the scheduler only if the task being suspended is the calling task.

### Resuming a Task, OSTaskResume() <a href="#taskmanagement-resumingatask-ostaskresume" id="taskmanagement-resumingatask-ostaskresume"></a>

As mentioned in the previous section, a suspended task can only be resumed by calling `OSTaskResume()`. The code for `OSTaskResume()` is shown in Listing 4.16.

```
INT8U  OSTaskResume (INT8U prio)
{
#if OS_CRITICAL_METHOD == 3                      
    OS_CPU_SR  cpu_sr;
#endif
    OS_TCB    *ptcb;
 
 
#if OS_ARG_CHK_EN > 0
    if (prio >= OS_LOWEST_PRIO) {                                     (1)
        return (OS_ERR_PRIO_INVALID);
    }
#endif
    OS_ENTER_CRITICAL();
    ptcb = OSTCBPrioTbl[prio];
    if (ptcb == (OS_TCB *)0) {                                        (2)
        OS_EXIT_CRITICAL();
        return (OS_ERR_TASK_RESUME_PRIO);
    }

    if ((ptcb->OSTCBStat & OS_STAT_SUSPEND) != OS_STAT_RDY) {         (3)
        ptcb->OSTCBStat &= (INT8U)~(INT8U)OS_STAT_SUSPEND;    
        if ((ptcb->OSTCBStat & OS_STAT_PEND_ANY) == OS_STAT_RDY) {    (4)
            if (ptcb->OSTCBDly == 0u) {                               (5)
                OSRdyGrp               |= ptcb->OSTCBBitY;            (6)
                OSRdyTbl[ptcb->OSTCBY] |= ptcb->OSTCBBitX;
                OS_EXIT_CRITICAL(); 
                if (OSRunning == OS_TRUE) {
                    OS_Sched();                                       (7)
                }
            } else {
                OS_EXIT_CRITICAL();
        } else {
            OS_EXIT_CRITICAL();
        }
        return (OS_ERR_NONE);
    }
    OS_EXIT_CRITICAL();
    return (OS_ERR_TASK_NOT_SUSPENDED);
}
```

(1) Because `OSTaskSuspend()` cannot suspend the idle task, it must verify that your application is not attempting to resume this task. Note that this test also ensures that you are not trying to resume `OS_PRIO_SELF` (`OS_PRIO_SELF` is #defined to 0xFF, which is always greater than `OS_LOWEST_PRIO`), which wouldn’t make sense – you can’t resume _self_ because _self_ cannot possibly be suspended.

(2) & (3) The task to resume must exist because you will be manipulating its `OS_TCB` , and it must also have been suspended.

(4) `OSTaskResume()` removes the suspension by clearing the `OS_STAT_SUSPEND` bit in the `.OSTCBStat` field.

(5) For the task to be ready to run, the `.OSTCBDly` field must be 0 because there are no flags in `OSTCBStat` to indicate that a task is waiting for time to expire.

(6) The task is made ready to run only when both conditions are satisfied.

(7) Finally, the scheduler is called to see if the resumed task has a higher priority than the calling task.

### Getting Information about a Task, OSTaskQuery() <a href="#taskmanagement-gettinginformationaboutatask-ostaskquery" id="taskmanagement-gettinginformationaboutatask-ostaskquery"></a>

Your application can obtain information about itself or other application tasks by calling `OSTaskQuery()`. In fact, `OSTaskQuery()` obtains a copy of the contents of the desired task’s `OS_TCB`. The fields available to you in the `OS_TCB` depend on the configuration of your application (see `OS_CFG.H`). Indeed, because µC/OS-II is scalable, it only includes the features that your application requires.

To call `OSTaskQuery()`, your application must allocate storage for an `OS_TCB`, as shown in Listing 4.17. This `OS_TCB` is in a totally different data space from the `OS_TCBs` allocated by µC/OS-II. After calling `OSTaskQuery()`, this `OS_TCB` contains a snapshot of the `OS_TCB` for the desired task. You need to be careful with the links to other `OS_TCBs` (i.e., `.OSTCBNext` and `.OSTCBPrev`); you don’t want to change what these links are pointing to! In general, only use this function to see what a task is doing — a great tool for debugging.

```
void MyTask (void *pdata)
{
    OS_TCB  MyTaskData;
 
    pdata = pdata;
    for (;;) {
        /* User code                    */
        err = OSTaskQuery(10, &MyTaskData);
        /* Examine error code ..        */
        /* User code                    */
    }
}
```

The code for `OSTaskQuery()` is shown in Listing 4.18.

```
INT8U  OSTaskQuery (INT8U prio, OS_TCB *pdata)
{
#if OS_CRITICAL_METHOD == 3                      
    OS_CPU_SR  cpu_sr;
#endif
    OS_TCB    *ptcb;
 
 
#if OS_ARG_CHK_EN > 0u
    if (prio > OS_LOWEST_PRIO) {                                     (1)
        if (prio != OS_PRIO_SELF) {
            return (OS_ERR_PRIO_INVALID);
        }
    }
    if (p_task_data == (OS_TCB *)0) {           
        return (OS_ERR_PDATA_NULL);
    }
#endif

    OS_ENTER_CRITICAL();
    if (prio == OS_PRIO_SELF) {                                      (2)
        prio = OSTCBCur->OSTCBPrio;
    }
    ptcb = OSTCBPrioTbl[prio];
    if (ptcb == (OS_TCB *)0) {                                       (3)
        OS_EXIT_CRITICAL();
        return (OS_ERR_PRIO);
    }
    if (ptcb == OS_TCB_RESERVED) {               
        OS_EXIT_CRITICAL();
        return (OS_ERR_TASK_NOT_EXIST);
    }

    OS_MemCopy((INT8U *)p_task_data, (INT8U *)ptcb, sizeof(OS_TCB)); (4)

    OS_EXIT_CRITICAL();
    return (OS_ERR_NONE);
}
```

(1) Note that I allow you to examine ALL the tasks, including the idle task. You need to be especially careful not to change what `.OSTCBNext` and `.OSTCBPrev` are pointing to.

(2) & (3) As usual, `OSTaskQuery()` checks to see if you want information about the current task and that the task has been created.

(4) All fields are copied using the assignment shown instead of field by field.
