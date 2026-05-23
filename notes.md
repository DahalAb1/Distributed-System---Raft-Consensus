In Search of an Understandable Consensus Algorithm

Raft was created to improve understandability for consensus algorithm. Previous we used Paxios which had a complex architecture and was hard to implement as well as learn, therefore, Raft was built to solve that problem. 

Raft applies verious techniques to improve understandability, 
    1. Decomposition (Raft separates leader election, log replication, and safety)
    2. State space reduction (Relative to Paxos Raft reduces the degree of nondeterminism and the ways server can be in consistent with each other.)


Consensus algorithms for practical systems typically
have the following properties:
• They ensure safety (never returning an incorrect result) under all non-Byzantine conditions, including
network delays, partitions, and packet loss, duplication, and reordering.
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