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



Jun 24 

Heartbeats and log-accept are separate. In AppendEntries, for Election, I had intially assumed AppendEntriesReply's Success variable, which is a boolean is supposed to denote if the heartbeat was success or not, this understanding was wrong. Success is used to determine if the log index of the leader and the follower matches or not. 

I've written some code in raft.go, I don't want to forget my changes for next session therefore I did not commit it. 


June 27 

One deep perspective change made my life easier when thinking about this consensus machine. Think of each global variable as preserved state, and think of each value inside the function as object state that's temporary. More specifically, think fo the go routines that ran as temporary states that perform some computation on the global states, and that we are performing work on the raft struct's global state, for both tbe server and the follower. 

This perspective shift, and closely looking into figure 2 helped me figure out how we can send AppendRPC from the leader update the nextIndex, and commit index. Also, I noticed, which I should have had before, when writing code for appendRPC's or server we should look into whatever's written in figure 2 for server Rules For server, for append RPC's Reciever Implementations. Just like how I took reference to fill up the structs and rpcs for election do the same in recpliation as well. Closely following figure 2 will reveal the answer by itself. 

A really signficant move here: So I moved the AppendRPC's definition, inside the peer lookup loop, instead of outside the loop which I had assumed would help me send empty append rpc, now instead of empty data we are sending data that's filled up with information, Entries, etc, and a single AppendRPC state can't define it, therefore each outside server will have extra information and will require unique appendRPC. Moving this was a significant step to make log replication to work. 

Important note : Raft uses the voting process to prevent a candidate from
winning an election unless its log contains all committed
entries.The
RequestVote RPC implements this restriction: the RPC
includes information about the candidate’s log, and the
voter denies its vote if its own log is more up-to-date than
that of the candidate.

Implemented applyCh in Make(), the server loads up another go routine that constantly checks if anything can be committed, Then we apply the changes into the applyCh channel, which takes in raftapi.ApplyMsg{} struct, the instructions for implementation is listed in raftapi/raftapi.go, and I simply followed that, created a new go routine that checks for committed data and then implemented that committed data into the state machine by sending the information to the server through applyCh channel. 


The stale reply bug (a reply from my own past)

While scanning my AppendEntries reply handler I got stuck on something that didn't sit right with me. The goroutine that handles a reply does this: it sends the RPC, and then LATER grabs the lock and updates nextIndex/matchIndex for that peer. The only safety check I had was `if res.ReplyTerm > rf.currentTerm` -> step down and return. I genuinely thought that was enough. My argument was: the greater term always wins, so if anything is wrong with terms, that check catches it and I step down.

But that check only protects me against replies that are AHEAD of me. It does nothing about a reply coming from my OWN past. That was the part I was missing.

Here is the timeline that finally made it click:

1. I'm the leader in term 3. I send an AppendEntries to peer P. The network delays it.
2. I get partitioned, lose leadership. Time passes. Eventually I rejoin and win a new election -> now I'm leader again, but in term 5. On winning, my nextIndex/matchIndex got freshly reinitialized.
3. That OLD reply from term 3 finally lands. res.ReplyTerm = 3.
4. My check: is res.ReplyTerm(3) > rf.currentTerm(5)? NO. So no step down, no return.
5. It falls straight through and edits my fresh term-5 nextIndex/matchIndex using the stale term-3 request. I just applied an old answer to my current state.

So "am I still the same leader" doesn't mean "am I a leader" — it means "am I still the same leadership INCARNATION that sent this exact request, i.e. the same term." I can be leader again in a different term with reset state, and the old reply will quietly corrupt it. The `>` check guards the future; it can't guard the past because the stale reply has a LOWER term, not a higher one.

A second, related symptom is matchIndex going backwards. matchIndex is supposed to only ever increase. But replies can arrive out of order:
- tick 1: send a request covering up to index 5
- tick 2: send a request covering up to index 8
- reply for tick 2 arrives first -> matchIndex = 8 (correct)
- reply for tick 1 arrives later -> my old line `rf.matchIndex[p] = req.PrevLogIndex + len(req.Entries)` sets it back to 5 (wrong)
Now my commit scan thinks P only has up to index 5 and can fail to commit index 8.

But wait — how does this situation even arise? I was lost on this for a while. Two replies in flight from the SAME peer at the same time felt impossible until I traced where the requests come from. My ticker loops every ~100ms, and EACH loop, for each peer, it spawns a brand new goroutine that fires one AppendEntries (the `go func...` line). It does NOT wait for the previous goroutine to finish. So:
- Tick 1 (t=0ms): spawn goroutine A -> sends AppendEntries to peer P. A is now parked waiting on the network for P's reply.
- Tick 2 (t=100ms): the loop comes around again, spawns goroutine B -> sends ANOTHER AppendEntries to P. B is now also parked waiting.
Now two replies are in flight from P at once, one for A and one for B. They travel over labrpc's simulated lossy/delayed network, and nothing guarantees A's reply comes back before B's. If A's packet got delayed more than B's, B's reply lands first.
So that's the whole origin: fire-and-forget goroutines, one per tick, never waiting -> multiple overlapping RPCs to the same peer -> replies can finish out of order. The `max` just makes me immune to which reply wins the race: a smaller stale value can never pull matchIndex back down.

I asked whether I could fix this by being more careful with reinitialization, and the realization was: no, reinit can't fix it. Reinit happens the moment I win (term 5), and the stale reply arrives AFTER that, so it just clobbers the freshly-reinitialized state. Reinit timing is upstream of the problem; the corruption is downstream. The bug is fundamentally "a reply is only valid for the term it was requested in," so the fix has to encode exactly that.

The fix (two parts):
- term/role guard right after taking the lock, so I drop replies that no longer apply:
    if rf.role != Leader || rf.currentTerm != req.LeaderTerm { return }
  (req.LeaderTerm is the term I stamped on the request when I sent it. If it doesn't match my current term, the reply is from a past incarnation -> throw it away.)
- never let matchIndex regress:
    rf.matchIndex[p] = max(rf.matchIndex[p], req.PrevLogIndex+len(req.Entries))
    rf.nextIndex[p]  = rf.matchIndex[p] + 1

Interesting side note I want to remember: the max() on matchIndex actually neutralizes stale SUCCESS replies on its own (a smaller stale value loses to max), so the term guard is mostly there to kill stale FAILURE replies (Success == false -> nextIndex -= 1), which would otherwise spuriously decrement. Together they're what real Raft implementations do.

This one does NOT show up in the basic 3B test because the network is reliable and replies come back promptly and in order. It only bites in the unreliable / partition tests. So I'm noting it now so future-me doesn't rediscover it the hard way when those tests start failing.
applyCh is a unbuffered channel so until the server recieves it pauses the go routine, this could cause some problem in the future but I believe the tests will work now. 



June 29

I've utilized AI to see for any problems in raft.go and reasoned about why some problems are actually problems and some are not. This showed me that we should not rely on ai all the times. 

The one(10) test gives 2 seconds for all 5 servers to apply the first entry, else it fails.
A new leader can only commit an entry from its own term, not a past term (paper section 5.4.2).
So if the leader changes during those 2 seconds and no new command comes in, the old entry
never commits, never gets applied, and the test fails. Bugs 1 and 2 cause that leader change.


Bug 1 — new leader is slow to send its first heartbeat (and may send it with a stale term) (solved)

What fails:  one(10), flaky. The leader that accepted the command steps down mid-agreement.
Reason:      two timing flaws in the ticker's leader branch.
Paper:       not a paper rule, implementation timing bug. (Section 5.2 only says the leader sends
             heartbeats periodically to stop followers from timing out.)
Why (step by step):
   1. A candidate wins the election and sets role = Leader.
   2. It does NOT heartbeat right away. Control goes back to the top of the ticker loop:
      sleep 10ms, read role, then sleep ANOTHER 100ms. So ~110ms pass before the first heartbeat.
   3. During that 110ms the other servers hear nothing from the new leader, so their election
      timers keep counting down.
   4. One of them times out, raises its term, starts a fresh election.
   5. That higher term reaches the new leader, which steps down. No stable leader -> another
      election round. Leadership churns.
   6. Smaller second flaw: the heartbeat term is read ONCE before the 100ms sleep. If the leader
      stepped down during that sleep, it still sends the old term. Those heartbeats get rejected,
      so even when it does send, it fails to quiet the rivals.
Effect on one(10):
      the test called Start(10) on the node that was leader. That node steps down with entry 10
      only in its own log. The next leader is in a new term and cannot commit entry 10 (a past-term
      entry, section 5.4.2), and no new command arrives -> entry 10 is never applied -> test fails.
Fix:         move the 100ms sleep to the END of the leader branch so a fresh leader heartbeats
             immediately. Re-read `rf.currentTerm` and re-check `rf.role == Leader` under lock for
             each peer (break if no longer leader) so it never sends a stale-term heartbeat.

argument : the AI over reasoned for this part, and was correct. The lab explicitely mentioned that each second 10 appendRPC should arrive. I had applied the RPC's sleep in the beginning, this could start a election and disrupt the leader's AppendRPC. 


Bug 2 — every server uses the same 400ms election timeout (solved)
Fix:         updated the timeout logic, implemented randomized value for election timeout, from 300 ms to 500 ms. 
--> This was a incorrect wrong bug pinned by the AI. I had applied a layered approach for election timeout, first see if the leader is off for 400 seconds, and then start election after this 400 second plus 50 to 300 ms, whatever the random value is supposed to be. AI flagged my style issue as an error which is absolutely incorrect. I've realized that to code properly with AI you'd have to follow it's pattern and design. My perspective on AI has changed a bit, instead of giving us the answer it is a context driven copy paste machine. I believe this is the consequence of using RAG based models which returns similar information from vecoterized database to reduce ai's computational overhead. If this code was put into a raw AI model without any Rag tricks (which is useful but for my use I'm bitter against) the model would have pointed this as not an error. 


Bug 3 — AppendEntries truncates the log blindly  

What fails:  later 3B tests (Rejoin, Backup). Committed entries can disappear.
Reason:      `rf.log = append(rf.log[:PrevLogIndex+1], Entries...)` always cuts and replaces.

Paper:       Figure 2, AppendEntries rule 3 — delete only when an entry CONFLICTS.

Why:         a delayed or reordered RPC carrying fewer/older entries overwrites and deletes
             entries the follower already had, even committed ones.

My Argument Against this error : The above method preserve's the Leader's completeness, and I don't see a reason that the log entries of the followers has to be truncated, if it is consistent with leader's log then it is, anything that does not match leader's log we can simply change it, this makes communication between leader and follower faster and reduces unnecessary computation to go through the log in the follower. 



What fails:  unreliable / Backup 3B tests. A server panics, then the cluster freezes.

Reason:      `rf.nextIndex[p] -= 1` runs unconditionally on every failed reply; the stale-reply
             guard only checks term/role, not whether the reply still matches the current nextIndex.
Paper:       section 5.3 — leader decrements nextIndex and retries (but never past the start).
When it fires:
      needs a follower behind enough to reject at prevIndex >= 1 (reconnect/backtrack) AND two or
      more concurrent in-flight rejects. Common in unreliable/Backup, rare in reliable. So the
      single-threaded reasoning explains why it is RARE, not why it is impossible.
Effect:      the ticker goroutine panics and dies, the leader stops heartbeating, cluster wedges.


My argument  against this error (correct for the SEQUENTIAL case):
      nextIndex never drops below 1. The dummy entry at index 0 always stays, so when nextIndex=1
      we send prevIndex=0, the dummy always matches, AppendEntries returns true, and we stop
      decrementing. If replies are handled one at a time, the walk-down self-stops at 1.

AI's reply, Where it still breaks (CONCURRENCY):
      the ticker fires a new AppendEntries to each peer every ~110ms WITHOUT waiting for the
      previous reply. So two RPCs to the same behind-follower can be in flight at once, both
      built when nextIndex=2. Both get rejected, both run the decrement:
        reply A: 2 -> 1
        reply B: 1 -> 0
      The "success at nextIndex=1" never gets a turn, because reply B was SENT while nextIndex
      was still 2 and lands after A already moved it. nextIndex=0 -> next send uses prevIndex=-1
      -> rf.log[-1] panics.
My Argument: 
    I don't see why we'd recieve 2 different appendRPC's, unless one is reply from Stale leader 
    and another from the current leader. Can you Explain this concept more clearly 

AI's response: 
Your ticker sends one heartbeat round every ~110ms and does not wait for the reply (it spawns a goroutine and moves on). 
So if a reply takes longer than 110ms to come back (slow/delayed network), 
the next round fires while the previous is still in flight. 
Now two requests from the same leader are in transit at once.

Timeline (one leader, term T, follower F is behind so it rejects, RTT ≈ 150ms > 110ms interval):

t=0ms    leader loop: reads nextIndex[F]=2, builds R1 (prev=1), sends, moves on
t=110ms  leader loop: R1 reply NOT back yet, so nextIndex[F] still 2,
                       builds R2 (prev=1), sends, moves on
t=150ms  R1 reply lands: Success=false -> nextIndex[F]: 2 -> 1
t=260ms  R2 reply lands: Success=false -> nextIndex[F]: 1 -> 0   <-- over-decrement
t=370ms  leader loop: nextIndex[F]=0 -> prev=-1 -> rf.log[-1] PANIC


Conclusion: This makes perfect sense to apply the guard. 

Fix: 
Human's Fix : 
if res.Success  == false { 	
				if rf.nextIndex[p] > 1 { 
								rf.nextIndex[p] -= 1 
  } 
} 

Ai's Fix: 
if res.Success == false {
    if req.PrevLogIndex < rf.nextIndex[p] {   // I don't see the point to compare this again, the reason we entered the false condition is because this particular comparision failed in the follower
        rf.nextIndex[p] = max(1, req.PrevLogIndex) // this will break the code, PrevLogIndex that was sent back to us by the follower, the follower does not return it's updated log with req.PrevLogIndex, the leader updates rf.nextIndex's value and then sends another AppendRPC to the followr to verifiy if the current log is similar. Changing this to the AI's recommendation would require us to refactor the whole logic. 
    }
}

Conclusion: 