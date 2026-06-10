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



# Go: Structs, Methods, and Interfaces

A minimal reference. Go has no classes and no inheritance. These three pieces replace them.

---

## 1. Struct = data

A struct holds fields. It is just grouped data.

```go
type Dog struct {
    name string
    age  int
}
```

---

## 2. Methods = behavior attached to a struct

A method is a function with a **receiver** — the `(d *Dog)` part. The receiver attaches the function to a type and acts as Go's explicit `this`.

```go
func (d *Dog) Bark() {
    d.name = "Rex"   // d refers to the specific Dog this was called on
}
```

Reading `func (d *Dog) Bark()`:
- `Dog` — the type this method belongs to.
- `d` — the name used inside the method to refer to the instance (your `this`). Pick any name; convention is a short one, reused on every method of that type.
- `*` — **pointer receiver**: the method can modify the struct and changes persist. Without `*` (value receiver), the method gets a *copy* and changes vanish on return.

Rule of thumb: use pointer receivers (`*`) when the method mutates state. If your updates mysteriously don't stick, a missing `*` is the cause.

**Call methods on instances, never on the type:**

```go
d := Dog{name: "Rex", age: 3}
d.Bark()      // correct — d is an instance
Dog.Bark()    // wrong — Dog is the type (the blueprint)
```

**Capitalization = access control** (no `public`/`private` keyword):
- Capital first letter → exported (public): `Bark`, `Make`
- lowercase first letter → unexported (private to the package): `name`, `age`

---

## 3. Interface = a checklist of methods

An interface lists method signatures. It is a requirement *for types*, used *by* functions.

```go
type Speaker interface {
    Speak() string
    Volume() int
}
```

This says: "to be a `Speaker`, a type must have both a `Speak() string` method and a `Volume() int` method."

### The key rule: satisfaction is implicit

A type satisfies an interface **automatically** by having the required methods. There is **no `implements` keyword**, and **nothing is written inside the struct**.

```go
type Dog struct {
    name string
}

func (d Dog) Speak() string { return "Woof" }
func (d Dog) Volume() int   { return 10 }

// Dog now satisfies Speaker — purely because it has both methods.
// The Dog struct never mentions Speaker.
```

Contrast with C++ (`class Dog : public Speaker`): C++ makes you *declare* the relationship. Go just looks at the methods and concludes it. This is the one feature that feels backwards coming from C++.

All-or-nothing: miss even one required method and the type does **not** satisfy the interface.

```go
var s Speaker = Dog{}   // compiles only if Dog has BOTH Speak() and Volume()
```

### How an interface is used

A function asks for the interface; you pass any type that satisfies it.

```go
func makeItSpeak(s Speaker) {
    fmt.Println(s.Speak())
}

makeItSpeak(Dog{})   // works if Dog satisfies Speaker
makeItSpeak(Cat{})   // works if Cat satisfies Speaker
```

The function says "I accept anything that can do these things" without naming specific types. That is Go's polymorphism — through capabilities, not a base class.

### Interfaces hold only methods, never fields

Everything inside an interface must have `()` — it is all methods. A bare field is illegal.

```go
type Speaker interface {
    Speak() string
    Age()   int     // OK — a method returning int
    age     int     // ILLEGAL — a field
}
```

If you want an int as **data**, it goes in the struct. If you want the interface to require "this type can give me an int," make it a **method** that returns int (`Age() int`). The interface never stores the int — it only demands a method that produces one.

### The empty interface: `interface{}` / `any`

An interface with zero methods. Since nothing is required, **every type satisfies it** — so it holds *any* value. It is Go's `void*` / `Object`.

```go
var x interface{}
x = 42        // fine
x = "hello"   // fine
x = Dog{}     // fine
```

`any` is the modern alias — identical to `interface{}`, just cleaner spelling.

```go
func Start(command any) (int, int, bool) { ... }
```

Here `command` is a parameter of type `any`, so `Start` accepts a command of any type. (This is exactly Raft's `Start` — the log stores arbitrary commands, so the type is "anything.")

---

## Syntax notes

**Return-value parentheses are optional for a single return, required for multiple:**

```go
Speak() string             // normal
Speak() (string)           // same thing — parens do nothing here
Speak() (string, error)    // parens REQUIRED — two return values
```

---

## One-line summary

| Concept   | Is...                            | Holds / Lists      |
|-----------|----------------------------------|--------------------|
| Struct    | data (a noun)                    | fields             |
| Method    | behavior attached via a receiver | logic              |
| Interface | a capability (a checklist)       | method signatures  |

- No classes — struct + methods is your class.
- No inheritance — embedding (struct-inside-struct) is the closest substitute.
- Interface satisfaction is implicit — having the methods is enough; nothing goes in the struct.
- `*` receiver to mutate; capital letter to export; `()` means it's a method.



Make notes no these later 
- Going through in search of learning consensus algorithm Section 5.2 (they decided on ranking candidates, but had lots of edge cases, so fell back to random timeout option as it was obvious and understandable.)
- 


// to run tests for lab, raft1
// to test everything in raft_test.go make raft1

# Run a single test
  cd /Users/abhineshdahal/Documents/Raft\ Consensus/6.5840/src/raft1
  go test -run TestInitialElection3A -v

  # Run all 3A tests
  go test -run 3A -v

  # Run all 3B tests
  go test -run 3B -v

  # Run everything
  go test -v


-- 

  ## Raft_tests.go : 

  TestInitialElection3A 


---

## June 4, 2026 — Raft 3A Debugging Session

**Goal:** Get `TestInitialElection3A` to pass.

**Left off:** About to implement `AppendEntries` handler — it needs to update `rf.lastHeard` when a heartbeat arrives so the follower doesn't trigger a spurious election.

**Bugs found (none fixed yet):**
1. `RequestVote` line 177 — condition inverted. Rejects vote when candidate has higher term, should grant it.
2. `AppendEntries` handler line 164 — empty, never updates `rf.lastHeard`.
3. `ticker()` line 313 — counts any successful RPC as a vote, should check `res.VoteGranted == true`.
4. `ticker()` — no locks. Reads/writes `rf.role`, `rf.currentTerm`, `rf.votedFor`, `rf.lastHeard` without holding `rf.mu`. Race detector catches this.
5. Election timeout line 285 — 5 seconds is too long, needs to be 150-300ms.

**Key insight:** Follower logic does NOT go in `ticker()`. It goes in the RPC handlers (`AppendEntries`, `RequestVote`). The RPC framework calls those automatically when a remote peer invokes them via `Call()`.

**Why errors change between runs without code changes:** Tests use randomness and goroutines — multiple bugs exist at the same time and which one surfaces first is non-deterministic.

Problem I saw past, AppendRPC is sent to all the servers, but I did not think of updating the server's term, this had started multiple elections even though everything was working, so just updating the value 
rf.lastHeard = time.Now(), fixed the problem, so the follower updates it's current time, that means it heard from the Leader, and does not start election. 

**June 8, 2026 — Insight: goroutine reply isolation**

**What I got wrong / was confused about:**
I thought passing `res RequestVoteReply` by value to the goroutine was a problem — "the response won't be updated if it's a copy." I wanted to pass a pointer (`*resRPC`) so the RPC could write back into it.

That instinct was backwards. Passing a pointer to the *shared outer variable* is exactly the bug — all goroutines would write their replies into the same memory at the same time → data race.

**Why value copy is correct:**

```go
go func(i int, req RequestVoteArgs, res RequestVoteReply) {
    rf.sendRequestVote(i, &req, &res)  // RPC writes into this goroutine's own res
    if res.VoteGranted { ... }         // safe — nobody else touches this res
}(i, reqRPC, resRPC)
```

- Each goroutine gets its own `res` copy on its stack. Totally isolated.
- `&res` gives the RPC a pointer to *that goroutine's private copy* — so the reply does land correctly.
- The old bug was `resRPC.VoteGranted` (outer shared var) — all goroutines were reading the same stale value, not their own reply.

**Rule:** when N goroutines each need their own reply buffer, pass by value so each gets a private copy. Only share state under a lock.

---

**June 8, 2026 — Insight: don't forget to count your own vote**

**What I got wrong:**
I initialized `majorityServers := 0` and used `numPeers - 1` to adjust the threshold. The math looked right on the surface but was off — for 3 peers it required both other peers to vote yes, which is too strict.

**Why it's wrong:**
The paper says a candidate votes for itself first, then sends RequestVote to others. The self-vote is real — it counts toward majority. `majorityServers` only collects replies from other peers via RPC, so it starts one vote short unless you pre-count yourself.

**Fix:**
```go
majorityServers := 1          // self-vote already cast (rf.votedFor = rf.me)
numPeers := len(rf.peers)     // includes self, e.g. 3

// after collecting peer replies:
if majorityServers > numPeers/2 {  // 3 peers: need >1 → 2 total votes
    rf.role = Leader
}
```

For 3 peers: start at 1, get 1 peer vote → total 2 → `2 > 1` → win. Correct.

**Rule:** initialize vote counter to 1 (self), not 0. The loop only collects peer responses — your own vote is already in.

---

**June 8, 2026 — Insight: WaitGroup Add/Done must always pair — skip before Add, not after**

**What I was confused about:**
I wanted to skip `i == rf.me` inside the goroutine body using `continue`. Thought it would just skip that iteration.

**Why that's wrong:**
`continue` inside the goroutine body doesn't skip the loop — it's already inside the goroutine. And even if it did skip, `wg.Add(1)` already ran. The counter is incremented but `wg.Done()` never fires → `wg.Wait()` blocks forever → deadlock.

Note: `defer wg.Done()` *would* still fire on goroutine exit (defers always run). But the issue is calling `continue` *in the for loop body after `wg.Add(1)`* — goroutine is spawned but immediately returns without doing work, while the Add already happened.

**Fix — skip before Add:**
```go
for i := range rf.peers {
    if i == rf.me {
        continue        // skip BEFORE wg.Add — no Add, no goroutine, no imbalance
    }
    wg.Add(1)
    go func(i int, ...) {
        defer wg.Done()
        ...
    }(i, ...)
}
```

**Rule:** `wg.Add(1)` and `wg.Done()` must always pair 1:1. If you want to skip an iteration, do it *before* `wg.Add` so the counter is never incremented for that case.



Another learning: Whenever you are writing, or changing a state which 



June 9
I've found a bug in code's current state. I was using wait groups to wait for all the RequestVoteRPC's go routines to return their response. And then I used this response to change our Candidate to the Leader. This idea is wrong because Raft does not wait for all the responses from it's peers. A Candidate is only concerned with 50% of vote from  it's peers. Therefore, The bug I was having with multiple election is found. In the next session I'll simply implement the check fo majority servers using go channels and convert my Candidate when I have recieved majority vote from our Peers. 

I believe there's a fatal error in the implementation of this Lab. The rf.Peer holds all the servers, when we break the connection for more than 50% of servers including the leader, we would never be able to reach majority vote. Therefore, this will make our follower stay as followers forever, which is wrong. 


June 10

1. I implemented yesterday's concern. Which was about my old code handling election with deadlock and a single variable, I've utilized channels and the multi election problem was solved. Today, I also implemented leader droppping to follower when the leader sees append RPC from a leader with higher index, and reading online post about other people building raft got me aware of AppendRPC that we could encounter when the server's sleeping. 

Lab1 Done!! Now time for Lab 2, leader election. This was a really long journey, I had limited my source of truth to Raft's paper, and the code that was given to me, and through this process I've amassed alot of knowledge. I am still rust with go, as I could not properly code up the channel's code at the ticker follower -> candidate part, which I utilized AI's help to get code ideas. For this session I utilized AI as a mechanism to get information quickly, without using it to problem solve. This was a good session, and I really enjoyed this.

---

**June 10, 2026 (afternoon session) — election bug sweep, 7 bugs found and fixed**

**Bugs found in election logic:**

1. **Stale vote goroutines leaking into the next election.** Goroutines don't stop when the election loop exits — they finish their RPC later and send votes into a channel from an election that's already over. Fixed with a `done` channel: `close(done)` broadcasts "stop" to every goroutine at once, and each goroutine uses `select` to either deliver its vote or drop it if the election ended.

2. **Election wait loop could spin forever.** My old loop only exited on (a) majority reached or (b) demoted to Follower. Third case — majority simply unreachable (partition, dead servers) — neither fires, candidate hangs forever, no retry. Real Raft uses an election *timeout*: give up after ~300ms, let the ticker start a fresh election with a new term. This would have failed TestReElection3A.

3. **Busy-spin burning CPU.** Same loop had no sleep/block — checked `len(ch)` at 100% CPU. Replaced with blocking `select` on the vote channel + timeout channel.

4. **Candidate ignored higher term in vote replies.** Paper Figure 2: if a response carries term T > currentTerm, set currentTerm = T and convert to follower. My goroutine only looked at VoteGranted.

5. **Leader ignored higher term in heartbeat replies.** Same Figure 2 rule. Scenario: old leader partitioned, cluster elects new leader with higher term, partition heals — old leader's heartbeats get rejected with the higher term, but it threw replies away and kept thinking it was leader → two leaders. Also reset `lastHeard` on step-down so the demoted leader doesn't instantly start an election against the real one.

6. **Vote grant didn't reset election timer.** A follower that grants a vote should reset `lastHeard`, otherwise it may immediately start a rival election against the candidate it just voted for. (I caught and fixed this one myself.)

7. **Blind election start after random sleep.** Ticker checks the timer, sleeps 50-300ms, then started the election without re-checking. A heartbeat arriving during that sleep should cancel the election — added a re-check of `lastHeard` under lock, `continue` if a leader appeared.

**Final vote-counting structure:**

```go
votes := 1 // self-vote
won := false
timeout := time.After(300 * time.Millisecond)

countLoop:
for {
    select {
    case <-ch:        // a vote arrived — consume it, count it
        votes++
        if votes >= noPeers/2+1 {
            won = true
            break countLoop
        }
    case <-timeout:   // 300ms passed — give up, retry next tick
        break countLoop
    }
    // also exit if AppendEntries demoted us to Follower mid-count
}
```

**Go concepts learned today:**

- **`len(ch)` vs `cap(ch)`** — `len` = items currently buffered, `cap` = buffer size. But receiving with `<-ch` (which consumes) is more reliable for counting than polling `len`.
- **Channel ≠ slice.** Channel is a communication pipe between goroutines; slice is a data structure. `len` works on both but they're different animals.
- **`done` channel pattern** — `make(chan struct{})`, then `close(done)` as a broadcast stop-signal. Works because a *closed* channel is permanently readable: every `<-done` returns immediately. `struct{}` is zero bytes — pure signal, no data.
- **Channel internals** — runtime `hchan` struct has a `closed` flag and a `recvq` of parked goroutines; `close()` sets the flag and wakes the entire recvq at once. That's the broadcast mechanism.
- **`select`** — blocks until ANY case is ready, takes whichever fires first. Votes race against the clock: vote first → count it; timeout first → quit.
- **`time.After(d)`** — returns a channel that receives one value after duration d. A timer disguised as a channel, so it composes with `select`.
- **Labeled break (`countLoop:`)** — `break` inside `select` only breaks the select, not the surrounding loop. Label the loop and `break countLoop` to exit the loop itself.

**Rule of the day:** every RPC reply must be checked for a higher term. Figure 2 says so and both step-down bugs (4, 5) were the same miss in two places.
