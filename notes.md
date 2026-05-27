In Search of an Understandable Consensus Algorithm

Raft was created to improve understandability for consensus algorithm. Previous we used Paxios which had a complex architecture and was hard to implement as well as learn, therefore, Raft was built to solve that problem. 

Raft applies verious techniques to improve understandability, 
    1. Decomposition (Raft separates leader election, log replication, and safety)
    2. State space reduction (Relative to Paxos Raft reduces the degree of nondeterminism and the ways server can be in consistent with each other.)


Consensus algorithms for practical systems typically
have the following properties:
• They ensure safety (never returning an incorrect result) under all non-Byzantine conditions, including
network delays, partitions, and packet loss, duplication, and reordering.https://go.dev/doc/effective_go#Getters
• They are fully functional (available) as long as any
majority of the servers are operational and can communicate with each other and with clients. Thus, a
typical cluster of five servers can tolerate the failure
of any two servers. Servers are assumed to fail by
stopping; they may later recover from state on stable
storage and rejoin the cluster.
• They do not depend on timing to ensure the consistency of the logs: faulty clocks and extreme message
delays can, at worst, cause availability problems.
• In the common case, a command can complete as
soon as a majority of the cluster has responded to a
single round of remote procedure calls; a minority of
slow servers need not impact overall system performance.



Notes on Go

Formatting --> As an example, there's no need to spend time lining up the comments on the fields of a structure. Gofmt will do that for you. 

Getters --> Go doesn't provide automatic support for getters and setters. There's nothing wrong with providing getters and setters yourself, and it's often appropriate to do so. 

owner := obj.Owner() // getter 
if owner != user {
    obj.SetOwner(user) // setter 
}

Data --> Go has two allocation primitives, the built-in functions new and make. They do different things and apply to different types, which can be confusing, but the rules are simple. 


### Concurrency




Goroutines --> They're called goroutines because the existing terms—threads, coroutines, processes, and so on—convey inaccurate connotations. A goroutine has a simple model: it is a function executing concurrently with other goroutines in the same address space. It is lightweight, costing little more than the allocation of stack space. 

go list.Sort()

func Announce(message string, delay time.Duration) {
    go func() {
        time.Sleep(delay)
        fmt.Println(message)
    }()  // Note the parentheses - must call the function.
}

Channels
Like maps, channels are allocated with make, and the resulting value acts as a reference to an underlying data structure. If an optional integer parameter is provided, it sets the buffer size for the channel. The default is zero, for an unbuffered or synchronous channel.

ci := make(chan int)            // unbuffered channel of integers
cs := make(chan *os.File, 100)  // buffered channel of pointers to Files


Note: 
A semaphore is a counter in memory that limits how many workers can run a section of code at once.A semaphore is a counter in memory that limits how many workers can run a section of code at once. Workers grab a slot before entering, release it after. If all slots are taken, others wait

