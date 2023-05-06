Download Link: https://assignmentchef.com/product/solved-ecse427-assignment-2-simple-user-level-thread-scheduler
<br>
In this project, you will develop a simple many-to-many user-level threading library with a simple first come first serve (FCFS) thread scheduler. The threading library will have two types of executors – the executors are kernel-level threads that run the user-level threads. One type of executor is dedicated to running compute tasks and the other type of executors is dedicated for input-output tasks.

The user-level threads are responsible for running the tasks, which are C functions. Your threading library is expected to at least run the tasks we provide as part of this assignment. A task is run by the scheduler until it completes or yields. The tasks we provide here do not complete (i.e., they have <strong>while (true)</strong> in their bodies). The only way a task can stop running once it is started is by yielding the execution. A task calls <strong>sut_yield()</strong> for yielding (i.e., pausing) its execution. A task that is yielding is put back at the end of the task ready queue. Once the running task is put back in the ready queue, the task at the front of the queue is selected to run next by the scheduler.

A newly created task is added to the end of the task ready queue. To create a task we use the <strong>sut_create()</strong> function, which takes a C function as its sole argument. When the task gets to run on the executor, the user supplied C function is executed.

All tasks execute in a single process. Therefore, the tasks share the process’ memory. In this simple user-level threading system, variables follow the C scoping rules in the tasks. You can make variables local to a task by declaring them in the C function that forms the “main” of the task. The global variables are accessible from all tasks.

The <strong>sut_yield()</strong> pauses the execution of a thread until it is selected again by the scheduler. With a FCFS scheduler the task being paused is put at the back of the queue. That means the task would be picked again after running all other tasks that are ahead of it in the queue. We also have a way of terminating a task’s execution by calling <strong>sut_exit()</strong>, which stops the execution of the current task like <strong>sut_yield()</strong> but does not put it back into the task ready queue for further execution.

So far, we described what happens with the compute tasks. The threading library has two types of executors. One type is dedicated for compute tasks and is responsible for carrying out all that is described above. The other type of executor is dedicated for input-output (I/O). For instance, a task would want to write a block of data to a file or read a block of data from a file. This is problematic because input-output can be blocking – that is the thread is blocking until the disk data is fetched. We have a problem when a user-level thread blocks – the whole program would stall. To prevent this problem, we use a dedicated executor for I/O. The idea is to offload the input and output operations to an I/O executor such that the compute executor would not block.

For instance, <strong>sut_read()</strong> would read data from the disk much like the <strong>read()</strong> you would do. The <strong>sut_read()</strong> would not have the data it is trying to retrieve for a while because the disk is slow compared to the CPU. To prevent the whole application stalling, we want to receive the data without stalling the executor with the following approach. When a task issues <strong>sut_read()</strong> we send the request to a request queue and the task that invoked the <strong>sut_read()</strong> itself is put in a wait queue. When the response arrives, the task is moved from the wait queue to the task ready queue and it would get to run in a future time. We have <strong>sut_write()</strong> to write data to the file. We follow the same approach and make write non-blocking as well. The asynchronous writing offered by <strong>sut_write() </strong>need not give the same performance benefit as <strong>sut_read() </strong>because the writes could be just writing to the kernel buffer instead of the disk in the OS provided implementation. Same way we also have <strong>sut_open()</strong> and expect the I/O executor to make the connection to the file (i.e., open the file).




<h1>Overall Architecture of the SUT Library</h1>

The simple user-level threading (SUT) library that you are developing in this assignment has the following major components. In the following discussion we assume that we have one compute executor (C-EXEC) and one I/O executor (I-EXEC). This restriction is relaxed in the last part of the assignment. The C-EXEC is responsible for most the activities in the SUT library. The I-EXEC is only taking care of the I/O operations. Creating the two kernel-level threads to run C-EXEC and I-EXEC, respectively is the first action performed while initializing the SUT library.

The C-EXEC is directly responsible for creating tasks and launching them. Creating a task means we need to create a task structure with the given C task-main function, stack, and the appropriate values filled into the task structure. Once the task structure is created, it is inserted into the task ready queue. The C-EXEC pulls the first task in the task ready queue and starts executing it. The executing task can take three actions that can alter its state:

<ol>

 <li>Execute <strong>sut_yield()</strong>: this causes the C-EXEC to take over the control. That is the user task’s context is saved in a task control block (TCB) and we load the context of C-EXEC and start it. We also put the task in the back of the task ready queue.</li>

 <li>Execute <strong>sut_exit()</strong>: this causes the C-EXEC to take over the control like the above case. The major difference is that TCB is not updated or the task inserted back into the task ready queue.</li>

 <li>Execute <strong>sut_open()</strong>: this causes the C-EXEC to take over the control. We save the user task’s context in a task control block (TCB) and we load the context of C-EXEC and start it, which would pick the next user task to run. We put the current task in the back of the wait queue. The I-EXEC would execute the function that would actually open the file and the result of the open would be returned by the <strong>sut_open()</strong> The result of the open is an integer much like the file descriptor returned by the OS. You can use the OS file descriptor or map it to another integer. If a mapping is done, a mapping table needs to be maintained inside the I-EXEC.</li>

 <li>Execute <strong>sut_read()</strong>: this causes the C-EXEC to take over the control. We save the user task’s context in a task control block (TCB) and we load the context of C-EXEC and start it, which would pick the next user task to run. We put the current task in the back of the wait queue. The I-EXEC thread is responsible for reading the data from the file. Once the data is completely read and available in a memory area that is passed into the function by the calling task. That is, the I-EXEC does not allocate memory – it uses a memory buffer given to it. We assume the memory buffer provided by the calling task is big enough for the data read from the disk.</li>

 <li>Execute <strong>sut_write()</strong>: this causes the C-EXEC to take over the control like the above calls. The I-EXEC thread is responsible for writing the data to the file. It uses the file descriptor (fd) to select the file that needs to receive the data. The contents of the memory buffer passed into the <strong>sut_write()</strong> is emptied into the corresponding file.</li>

 <li>Execute <strong>sut_close()</strong>: this causes the C-EXEC to take over the control like the above calls. The I-EXEC thread is responsible for closing the file. We need not have done file closing in an asynchronous manner – so it is done to keep all IO calls asynchronous.</li>

</ol>

After the SUT library is done with the initializations, it will start creating the tasks and pushing them into the task ready queue. Once the tasks are created, the SUT will pick a task from the task ready queue and launch it. Some tasks can be launched at runtime by user tasks by calling the <strong>sut_create()</strong> function. The task scheduler just picks the task at the front of the queue and executes it. It might find that there are no tasks to run in the task ready queue. For instance, the only task in the task ready queue could issue a read and go into the wait queue. To reduce the CPU utilization the C-EXEC will go take a short sleeps using the nanosleep() command in Linux (a sleep of 100 microseconds is appropriate). After the sleep, the C-EXEC will check the task ready queue again.

The I-EXEC is primarily responsible for processing all IO functions. It can use message queues to implement its functionality. The actual details of how I-EXEC should implement its operations are left for you as a design exercise. Once the function execution is complete, the I-EXEC puts the task back in the ready queue so it can proceed with the computations.




The <strong>sut_shutdown()</strong> call is responsible for cleanly shutting down the thread library. We need to keep the main thread waiting for the C-EXEC and I-EXEC threads and it one of the important functions of <strong>sut_shutdown()</strong>. In addition, you can put any termination related actions into this function and cleanly terminate the threading library.




<h1>The SUT Library API and Usage</h1>

The SUT library will have the following API. You need to follow the given API so that testing can be easy.

<strong>void sut_init(); </strong>

This call initializes the SUT library. Needs to be called before making any other API calls.

<h2>bool sut_create(sut_task_f fn);</h2>

This call creates the task. The main body of the task is the function that is passed as the only argument to this function. On success this returns a True (1) else returns False (0).

<h2>void sut_yield();</h2>

In SUT a running task can yield execution before the completion of the function by issuing this yield call. Another task that is present in the ready queue will start running and the task running the yield will get enqueued at the back of the ready queue.

<h2>void sut_exit();</h2>

You call this function terminate the task execution. This should not terminate the whole program unless there is only one task. That is, if there are multiple SUT tasks running in the system, exiting a task using this call is just going to terminate that task.

<h2>int sut_open(char *fname);</h2>

This is an IO function. You are requesting the system to open the file specified by the name. If there is no such file, you will get a negative return value. Otherwise, a non-negative value will be returned.

<h2>void sut_write(int fd, char *buf, int size);</h2>

We write the bytes in buf to the disk file that is already open. We don’t consider write errors in this call.

<strong>void sut_close(int fd); </strong>

This function closes the file that is pointed by the file descriptor.

<h2>char *sut_read(int fd, char *buf, int size);</h2>

This function is provided a pre-allocated memory buffer. It is the responsibility of the calling program to allocate the memory. The size tells the function the max number of bytes that could be copied into the buffer. If the read operation is a success, it returns a non NULL value. On error it returns NULL.

<strong>void sut_shutdown(); </strong>




<h1>Context Switching and Tasks</h1>

You can use the <strong>makecontext()</strong> and <strong>swapcontext()</strong> to manage the user-level thread creation, switching, etc. The sample code provided in the <strong>YAUThreads</strong> package illustrates the use of the user-level context management in Linux. The intention of the sample code is to illustrate how you can implement user-level threads – it is not a starter code. You are given full permissions to reuse portions of the sample code as appropriate.




<h1>Important Assumptions</h1>

Here are some important assumptions you can make in this assignment. If you want to make additional assumptions, check with the TA-in-charge (Zhelin) or the professor.

<ul>

 <li>There are no interrupts in the user-level thread management to be implemented in this assignment. A task that starts running only stops for the reasons given in Section 2.</li>

 <li>You can use libraries for creating queues and other data structures – you don’t need to implement them yourself! We have already given you some libraries for implementing data structures.</li>

</ul>