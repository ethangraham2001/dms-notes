# Distributed Transactions 

## 2 Phase Commit

### Consensus Problem

The distributed consensus problem consists in reaching an agreement between all
working processes on the value of a variable. This is a hard problem in an
asynchronous distributed environment. Asynchronous means that no timing 
assumptions can be made about the speed of processes or the network delay - 
i.e. we can't distinguish between slow network and a system failure.

In the generals problem, which is just distributed consensus, I need to know:

- My own state
- The state of the other
- That the other knows my state
- That the other knows that I know their state
- That I know that the other knows that I know their state
- ...

If the system is entirely asynchronous, then we cannot solve the problem only 
by sending messages.

### Atomic Commitment

All sites must agree on whether to commit or abort a transaction, and all sites
must make the same decision. We want to enforce the following properties

- **AC1:** All processors that reach a decision reach the same one
    _(agreement, consensus)_
- **AC2:** A processor cannot reverse its decision
- **AC3:** Commit can only be decided if all processors vote YES _(no imposed
    decisions)_.
- **AC4:** If there are no failures and all processors voted YES, the decision
    will be to commit _(non triviality)_
- **AC5:** Consider an execution with normal failures. If all failures are
    repraired and no more failures occur for sufficiently long, then all
    processors will eventually reach a decision _(liveness)_.

### 2PC Protocol

In this protocol, we have a coordinator node, and participant nodes.

- Coordinator sends `VOTE-REQ` to all participants
- Upon receiving a `VOTE-REQ`, a participant sends a message with `YES` or 
    `NO`, sending `NO` if it aborts.
- The Coordinator collects all votes. If all are `YES`, then it can send a
    `COMMIT` to all, otherwise it sends `ABORT` to all that voted `YES`.
- A participant receiving `COMMIT` or `ABORT` from the coordinator decides
    accordingly and stops.

We now argue the correctness of the protocol by showing that it satisfies
conditions AC1-5.

- AC1: every processor decides what the coordinator decides
- AC2: any processor arriving at a decision stops.
- AC3: the coordinator will decide to commit if all decide to commit.
- AC4: If there are no failures and everybody voted yes, then the decision will 
    be to commit.
- AC5: the protocol needs to be extended in the case of failures. For example,
    if we timeout, a site may need to "ask around" neighbor nodes.

#### Timeout possiblilities

From the coordinator's perspective, we can only timeout in one place, which is
the phase in which we collect votes from all participant nodes. In the case of
a failure, we may block forever. However, in this case, the coordinator can 
just abort, as it knows that no participant will have committed here - thus
unblocking us by timing out.

From the participant perspective, it can block at two moments

- Wait for `VOTE-REQ`. If I haven't received one, then no one could have
    committed yet.
- Then when waiting for a decision after voting `YES`, I can block. I can't 
    abort, because I said I was going to commit. I can't commit, bceause 
    someone may have voted no. This is a problem - I can perhaps ask the 
    coordinator, but it may have failed. I can maybe ask the other participants
    about the coordinators decision. We can run what is called a _cooperative
    termination protocol_. This state is called **uncertainty period**.

![Timeout possibilities for coordinator node](images/2pc-coordinator.png)
![Timeout possibilities for a participant node](images/2pc-participant.png)

Since the coordinator has no uncertainty period, and there is always at least
one processor that has decided, then if all failures are repaired, all
processors eventually reach a decision.

If the coordinator fails after receiving all `YES` votes but before sending any
`COMMIT` message, then the participants are unable to decide on anything. This
is the blocking behavior of 2PC.

For recovery and persistence, we use a log record which requires flushing the
log buffer to disk I/O. This is done for every state change in the protocol,
and it is done for every distributed transaction. This becomes pretty 
expensive.

When sending a `VOTE-REQ`, the coordinator writes a `START-2PC` log record so
that we can identify the coordinator. If a participant votes `YES/NO`, it will
write a corresponding record in the log _before_ the vote (WAL fashion). If
the coordinator decides to commit or abort, then it writes a log record before
sending the message. After receiving a coordinator's decision, a participant
writes its own decision in the log.

We can run a variant of 2PC called **linear 2PC** which minimizes the number of
network messages sent by exploiting a particular linear network configuration.
The total number of messages will be $2n$ instead of $3n$, but we also require
more rounds, which is $2n$ instead of $3$.

### 3PC protocol

With the insight that 2PC may block if the coordinator fails after sending a
`VOTE-REQ` to all processes, and all processes vote `YES`, we can reduce the
vulnerability window even further by using a more complex protocol. This isn't
really used in practice because it is too expensive, and probability of 
blocking in 2PC is low enough that we prefer to simply use that. But it is a
good means for understanding subtleties of atomic commitment.

Look at 2PC once more. If a process fails and everybody is uncertain, then 
there is no way to know whether the process has committed or aborted. But if
everyone is uncertain, then it implies that everybody voted `YES`. However,
uncertain processes cannot make a decision - just because we are uncertain 
doesn't mean that everybody else is.

3PC enforces the NB rule _(non-blocking rule)_. No operational process can 
decide to commit if there are operational processes that are uncertain. The NB
rule guarantees that if anybody is uncertain, then nobody can have decided to
commit, thus when running cooperative termination protocol, if a process finds
out that everybody else is uncertain, they can all safely decide to abort. This
now implies that the coordinator node cannot decide by itself as it did in 2PC.
In order to make a decision, we need to be sure that everybody is out of the
uncertainty period - therefore the coordinator must first tell all processes
what is going to happen _(request votes, prepare to commit, commit)_. This
implies another round of messages, hence the 3 in 3PC.


![3PC from coordinator's perspective](images/3pc-coordinator.png)
![3PC from participant's perspective](images/3pc-participant.png)

This is interesting as the processes know what will happen before it does. When
the coordinator reaches "bcast pre-commit" stage, it knows the decision will be
to commit. When a participant received pre-commit, it knows that the decision
will be to commit. While waiting for pre-commit, the participant knows that no
one has committed, and therefore there is no uncertainty period here.

The extra round of messages if used to spread knowledge across the system,
providing useful information about what is going on at other processes.

- If coordinator times out waiting for votes = ABORT.
- If participant times out waiting for vote-req = ABORT.
- If coordinator times out waiting for ACKs = ignore those who did not sent the 
    ACK! (at this
stage everybody has agreed to commit).
- If participant times out waiting for pre-commit = still in the uncertainty 
    period, ask around.
- If participant times out waiting for commit message = not uncertain any more 
    but needs to ask around!

Persistence in 3PC happens very similarly to 2PC, in that it uses a WAL.

#### Termination Protocol

- Elect a new coordinator
- New coordinator sends a `STATE-REQ` to all participants, who then send their
    state _(aborted, committed, uncertain, committable)_
- If some are aborted, abort. If some are committed, commit. If all uncertain,
    abort. If some committable but no committed received, send `PRE-COMMIT` to
    all and wait for ACKs to send a commit message.


```
TODO: FINISH THIS SECTION. NO ENERGY TO DO IT NOW, BUT RIGHT AT THE END OF 
SLIDES
```


