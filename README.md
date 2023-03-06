# Starvation Free Solution for Readers-Writers Problem

## Classical Solution Readers-Writers Problem

- To synchronize the reader's and writers’ access to the shared resource so that they can perform their roles, we can use various critical section solutions like semaphores or monitors.
- The classical solution of the readers-writers problem uses two semaphores
    - `write_mutex` → to control the mutual exclusion for the writing process
        - updated by the first reader to ensure that the writer doesn’t enter the critical section making any changes in the file/ database.
        - or is updated by the last reader leaving to signal the writer that there is no reader anymore; hence file can be edited by the writers.
    - `mutex` → this mutex controls the mutual exclusion for updating the `readers_count` variable, which tells a number of readers in the critical section and is shared.
        - this is updated while the `readers_count` is changed by the leaving or entering readers.
- The initial values are:
    - `readers_count` = 0
    - `mutex` = 1
    - `write_mutex` = 1
- This includes the following pseudocode for the writer process:
    
    ```jsx
    // Writer Process
    
    wait(write_mutex);
    
    	// Critical Section -> writing performed
    
    signal(write_mutex);
    ```
    
- The reader process has:
    
    ```jsx
    // Reader Process
    
    wait(mutex);    // as updating the readers count as reader entering
    readers_count++;
    if (readers_count == 1)
    	wait(write_mutex);
    	// as the first reader enters, decrements to deny access to writers
    signal(mutex);
    
    	// Critical section -> reading performed
    
    wait(mutex);
    readers_count--;
    if (readers_count == 0)
    	signal(write_mutex);
    	// as the last reader leaves signal to allow access to writers
    signal(mutex);
    ```
    
- This provides mutual exclusion and progress but leads to starvation. The main reason behind it was that:
    - if readers keep coming one after the other after the first reader has decremented the `write_mutex`, then the writer will not be allowed to access the resource.
    - Hence, leading to starvation of the writer process.
- Thus, to prevent starvation, there was a need for a mechanism to give equal priority to both, which would queue the readers if a writer asked for access to the critical section first.

## Implementation of the Process

The class `Process`, has been created for storing the basic information which a process holds, like `pID`, `state`, and a variable `role` is there to find that the process is a `reader` or a `writer` process.

```jsx
class Process
{
public:
    int pID;
    string state;
    string role;
    Process()
    {
        this->pID = 0;
        this->state = "initialized";
        this->role = "reader";
        readerProcess(this->pID);   // once the process is created it calls its respective function
    }
    Process(int pID, string role)
    {
        this->pID = pID;
        this->state = "initialized";
        this->role = role;
        if (role == "reader")
            readerProcess(this->pID);
        else
            writerProcess(this->pID);
    }
};
```

The processes are managed by a class `ProcessManager` which can provide the process on giving `pID` as a parameter to the `getProcess` method of its instance.

```jsx
class ProcessManager
{
public:
    vector<Process> processes;
    void addProcess(Process process)
    {
        processes.push_back(process);
    }

    Process getProcess(int pID)
    {
        for (int i = 0; i < processes.size(); i++)
        {
            if (processes[i].pID == pID)
            {
                return processes[i];
            }
        }
        return Process();
    }
};
```

Its instance is created so that the processes can be recognized with its `pID`.

```jsx
static ProcessManager pm;
```

The class `Semaphore` instantiates semaphores with a semaphore’s initial value and a queue that holds the process which tries to access it while the semaphore is not available; these are triggered by the signal process when the semaphore is released.

It includes processes of the semaphores `wait(int pID)` and `signal()`, which are implemented as follows:

```jsx
class Semaphore
{
public:
    int semaphoreValue;
    queue<Process> queueOfProcesses;

    Semaphore()
    {
        semaphoreValue = 1;
    }
    Semaphore(int val)
    {
        semaphoreValue = val;
    }

    void wait(int pID)
    {
        if (this->semaphoreValue <= 0)
        {
            pm.getProcess(pID).state = "blocked";
            queueOfProcesses.push(pm.getProcess(pID));
        }
        this->semaphoreValue--;
    }

    void signal()
    {
        this->semaphoreValue++;

        if (this->semaphoreValue > 0)
        {
            queueOfProcesses.pop();
            // And after poping the first process in the queue make a system call to the process so that it checks again
        }
    }
};
```

Here the `wait(int pID)` function takes the process ID as a parameter and checks the value of its semaphore instance. If the value is non-zero, then it shows that the process can access the semaphore, and hence then decrement, else the process is pushed to the queue of the semaphore, which shows the processes which are yet to access the semaphore.

The `signal()` function releases the value of its semaphore and removes the process from its queue, allowing it to access the semaphore.

We have: 

```jsx
// Shared variables
int readers_count = 0;   // number of readers accessing the critical section

// Semaphores defined
Semaphore mutex(1);       // mutual exclusion for shared variables
Semaphore write_mutex(1); // mutual exclusion for writers accessing the critical section
Semaphore access_mutex(1); // for preventing starvation of writers in the updated approach
```

Hence, using the above implementation, the processes can be formed by the user in the following manner for demonstration.

```cpp
// Reader process
    int pID = 1000;
    Process p1(pID, "reader"); // a reader process added
    pm.processes.push_back(p1);
    readerProcess(pID);
// Writer process
    int pID = 2000;
    Process p1(pID, "writer"); // a writer process added
    pm.processes.push_back(p1);
    writerProcess(pID);
```

### Initial Approach

- In the previous solution, we kept the `readers_count` variable to store the number of readers currently in the critical section.
- In my initial approach, I thought of introducing another variable, `waiting_writers` variable, to store the number of writers who arrived for access while the readers were accessing the critical section.
- This variable will be checked in the reader process. While the other reader process comes after the writer process it tries to access it but will be queued so that the current readers can complete their execution and release the `write_mutex` semaphore.
- This solution involved was similar to the classical, just with the addition of `waiting_writers` variable, where reader doesn’t access the `write_mutex` unless there are no waiting writers present, which is seen using the `waiting_writers` variable.
- However, if the reader and writer arrive at the same time, and suppose a writer starts accessing the critical section, but if by the time, the variable `waiting_writers` is not updated, then this would lead to access of the critical section by the reader as well.
- Thus giving the possibility of starvation by giving monopolized access if both arrive at the same time.

## Updated Starvation Free Solution

- In the previous approach, as the extra parameter to prevent starvation was variable, it had a loophole of being accessed by both at a time on arriving together.
- So, for giving both processes fair access to the critical section, I added a new semaphore variable `access_mutex` for controlling the order of the access of the critical section by the processes.
- This is acquired by both processes when they arrive. Hence, the problem of the classical solution is solved when readers keep coming after a writer is waiting.
    - This is because the prior acquisition of the `access_mutex` by the writer process makes it execute earlier than the later-arrived reader processes due to the FIFO queue of `access_mutex`
- Thus the check of `access_mutex` makes sure that the reader process doesn’t keep entering the critical section while the writer is waiting.
- As the parameter on which we are relying is not a variable but rather a semaphore, the problem rising while both arrive at the same time doesn’t occur here.

## Reader Process

- So now the reader process requires the `access_mutex` at the beginning, if it is not acquired by any other process it is acquired and follows the process as in the classical solution.
    - By acquiring the `mutex`, to update the `readers_count` and acquire the `write_mutex` while the reader is the first reader
- Then before entering the critical section, releases the `access_mutex` to look at the order of access by the readers and writers, whichever, arrives first.
    - And now can be accessed by any process arriving after it.
    - If the writer process acquires the `access_mutex`, the reader has to wait to acquire the same and thus not enter the critical section, while the `write_mutex` is also acquired by the first reader process.
- The updated reader process is as follows:
    
    ```jsx
    // Reader Process
    void readerProcess(int pID)
    {
        access_mutex.wait(pID);
    
    		mutex.wait(pID);
        readers_count++; // increment number of readers accessing the critical section
        if (readers_count == 1)
        {
           // being the first writer aquires the write_mutex
            write_mutex.wait(pID);
        }
        mutex.signal();
    
    		access_mutex.signal();
       
    		// critical section
    
        mutex.wait(pID);
        readers_count--;
        if (readers_count == 0)
        {
            // if this is the last reader, signal any waiting writers
            write_mutex.signal();
        }
        mutex.signal();
    }
    ```
    

## Writer Process

The process is similar to the classical solution with an addition of the check of the `access_mutex` semaphore for acquiring the `write_mutex`, which is done so that the writing process occurs by signaling the other process preventing them from performing any action on the critical section.

The updated writer’s process is as follows:

```cpp
// Writer Process
void writerProcess(int pID)
{
		access_mutex.wait(pID);

	  write_mutex.wait(pID);

		// before the process enters the critical section, releases the access_mutex to allow the other process to queue up for the access to critical section after it
		access_mutex.signal();
	  
        // critical section

    write_mutex.signal(); // release the write mutex
}
```

Hence, the above solution provides the starvation-free solution to the readers’-writers’ problem by satisfying mutual exclusion, progress, and bounded waiting benchmarks.