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



Jun 20 

For log replication, in the current version 


rf.Start() --> this takes user input  (command interface{}, where do we get command from?)
applyCh --> this is where we send committed logEntries 
appendRPC --> this is where we send fresh log entries to other followers 

updated understanding 
rf.Start() --> this takes user input, I misunderstood here, I thought this was supposed to be defined by my code, but this is invoked by the server. The server inputs the log if it's a leader by taking it from the user. I caught this when I did grep -rn ".Start(" 6.5840/src/raft1, which showed grep utilized in server.go, and 6.5840/src/raft1/server.go:89 returns rf.Start(command). 

```
Programmer's Note : If you want to find where the function, or certain implementation is used. You can use grep. 

These are the Core flags:
  - -r recursive (search dirs)
  - -n show line numbers
  - -i ignore case

  Example : grep -rn "applyCh" 6.5840/src/raft1   
```


Summary of today's learnings: 
  1. applyCh is local, not network — Go channel = goroutine→goroutine, same process. Two paths: peer↔peer via AppendEntries RPC (network, faked by labrpc), Raft→service via applyCh
  (top door). Confirmed by grep: tester reads my applyCh in server.go:108 (for m := range applyCh — applier loop).
  2. Start ≠ wait for commit — appends to leader log, returns (index, term, true) immediately. Replication + commit happen async in background. Don't block.
  3. ApplyMsg fields — CommandValid: true, Command, CommandIndex. Sent on every node when commitIndex > lastApplied.
  4. votedFor lifecycle — struct field. Set in RequestVote (grant vote), ticker (vote self), reset -1 on new term. AppendEntries never sets it to leader; track leader separately if
  needed.
  5. Heartbeat = empty AppendEntries — don't write 2 RPCs. One timer, entries = log[nextIndex[i]:]. Empty when follower caught up. Heartbeat is circumstance, not special code.
  6. nextIndex/matchIndex — per-follower arrays. i = follower index, NOT rf.me. nextIndex[rf.me] unused. Leader's own latest = len(rf.log).
  7. Go init gotcha — make([]int, n) zero-fills (not Python [0]*n). matchIndex starts 0; nextIndex starts len(log) (loop to set). Init in becomeLeader, not Make.




In the paper, figure 2, in the state part, and in Volatile state on leaders. I strongly believe keeping a matchIndex is enough, I don't see the reason why we'd have to give extra memory and store nextIndex, this feels redundant. 

I asked this question to AI (claude opus 4.8, effort : high) and it seems to agree: 

AI's short response: 
- matchIndex = needed. Drives commit. Must be true.
- nextIndex = optional. Just speed.
  
  You were right. One marker works. 

So from this I undersatnd that speed's the concern, but the extra memory usage just makes the process perform work more. 