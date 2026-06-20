Raft maintains the following properties, which together constitute the Log Matching Property:
    • If two entries in different logs have the same index and term, then they store the same command.
    • If two entries in different logs have the same index and term, then the logs are identical in all preceding entries.


    The leader maintains a nextIndex for each follower, which is the index of the next log entry the leader will
send to that follower. When a leader first comes to power, it initializes all nextIndex values to the index just after the
last one in its log (11 in Figure 7). If a follower’s log is inconsistent with the leader’s, the AppendEntries consistency 
check will fail in the next AppendEntries RPC. After a rejection, the leader decrements nextIndex and retries 
the AppendEntries RPC. Eventually nextIndex will reach a point where the leader and follower logs match. When
this happens, AppendEntries will succeed, which removes any conflicting entries in the follower’s log and appends
entries from the leader’s log (if any). Once AppendEntries succeeds, the follower’s log is consistent with the leader’s,
and it will remain that way for the rest of the term.

If desired, the protocol can be optimized to reduce the number of rejected AppendEntries RPCs. For example,
when rejecting an AppendEntries request, the follower can include the term of the conflicting entry and the first
index it stores for that term. With this information, the leader can decrement nextIndex to bypass all of the conflicting 
entries in that term; one AppendEntries RPC will be required for each term with conflicting entries, rather than one RPC per entry.

 Raft can accept, replicate, and apply new log entries as long as a majority of the servers are up; in the normal case a new entry
can be replicated with a single round of RPCs to a majority of the cluster; and a single slow follower will not
impact performance.

Raft uses the voting process to prevent a candidate from winning an election unless its log contains all committed
entries. A candidate must contact a majority of the cluster in order to be elected, which means that every committed
entry must be present in at least one of those servers.

The RequestVote RPC implements this restriction: the RPC includes information about the candidate’s log, and the
voter denies its vote if its own log is more up-to-date than that of the candidate. 

Raft determines which of two logs is more up-to-date by comparing the index and term of the last entries in the
logs. If the logs have last entries with different terms, then the log with the later term is more up-to-date. If the logs
end with the same term, then whichever log is longer is more up-to-date.

Raft never commits log entries from previous terms by counting replicas (majority of servers). Only log entries from the leader’s current
term are committed by counting replicas; once an entry from the current term has been committed in this way,
then all prior entries are committed indirectly because of the Log Matching Property. 

Interesting Find: 

Raft's algorithm is not robust to uncommitted entries. Even if one of the leader wins an election, and recieves a client request,
a uncommitted entries on a crashed leader just vanish, and the client only finds out by timing out and retrying — Raft guarantees no lost committed writes, 
not no lost attempts. For example, Raft promises if you saw "Success," it's permanent.  It does not promise your request will be remembered if you didn't see "Success." 
A timeout means "I don't know", and "I don't know" includes "your entry got silently thrown away when the leader crashed." You a user retries, and the new leader handles it.

