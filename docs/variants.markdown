---
layout: page
title: Variants
permalink: /variants/
---

There have been many Paxos variants introduced over the years. These
variants concentrate on making the basic Paxos algorithm more
efficient. We talk about some of these variants chronologically below
and point out the most important properties of each variant.

<img src="../static/variants.png" width="100%" usemap="#variantsmap" vspace="60">

<map name="variantsmap">
    <area shape="rectangle" coords="25,150,80,175" href="index.html" alt="Paxos">
    <area shape="rectangle" coords="220,11,310,28" href="#disk" alt="Disk Paxos">
    <area shape="rectangle" coords="245,35,365,55" href="#cheap" alt="Cheap Paxos">
    <area shape="rectangle" coords="290,65,380,80" href="#fast" alt="Fast Paxos">
    <area shape="rectangle" coords="320,90,455,105" href="#generalized" alt="Generalized Paxos">
    <area shape="rectangle" coords="405,35,620,55" href="#stoppable" alt="Stoppable Paxos">
    <area shape="rectangle" coords="525,60,600,75" href="#mencius" alt="Mencius">
    <area shape="rectangle" coords="450,90,655,115" href="#vertical" alt="Vertical Paxos">
    <area shape="rectangle" coords="635,40,800,100" href="#egalitarian" alt="Egalitarian Paxos">
</map>

<a name="disk"></a>
## Disk Paxos

Every replicated component in multi-decree Paxos is a process,
actively sending and receiving
messages.
[Disk Paxos](http://research.microsoft.com/en-us/um/people/lamport/pubs/disk-paxos.pdf)
is a variant of Paxos that replaces
acceptor processes with disks that support only read block and write
block operations to store the quorum state. This way, Disk Paxos does
not need separate acceptor processes.  In Disk Paxos, each leader owns
a block on every disk to which it can write its messages. To run phase
1 of the Synod protocol, a leader executes the following for each
disk. The leader writes a *p1a* message in its own block on the
disk and reads the blocks of other leaders on the same disk to check
if there is a **p1a** message with a higher ballot number. If the
leader does not discover a higher ballot number on a majority of
disks, its ballot is adopted. If it discovers a *p1a* message
with a higher ballot number, it starts over with a higher ballot
number. For phase 2, the leader repeats the same process
with **p2a** messages to determine if its proposals are accepted.

<a name="cheap"></a>
## Cheap Paxos

Multi-decree Paxos requires *2f+1* acceptors and *f+1*
replicas to tolerate *f* failures.
[Cheap Paxos](http://research.microsoft.com/en-us/um/people/lamport/pubs/web-dsn-submission.pdf) uses only *f+1* acceptors combined with
*f* cheap additional auxiliary acceptors that are kept idle during
normal execution, unless there is a failure. Cheap Paxos is a
variation of the reconfigurable multi-decree Paxos algorithm that
relies on the observation that a leader can send its p1a and p2a
messages to a fixed quorum of acceptors. As long as all acceptors in
that quorum are working and can communicate with the leaders, there is
no need for acceptors not in the quorum to do anything.

Cheap Paxos runs the multi-decree Paxos protocol with a fixed quorum
of *f+1* acceptors. Upon suspected failure of an acceptor in the fixed
quorum, an auxiliary acceptor becomes active so that there are once
again *f+1* responsive acceptors that can form a quorum. This new
quorum completes the execution of any outstanding ballots and
reconfigures the system replacing the suspected acceptor with a fresh
one, returning the system to normal execution.

<a name="fast"></a>
## Fast Paxos

In Paxos, each decision takes at least three message delays between
when a replica proposes a command and when some replica learns which
command has been chosen. Moreover, if there are multiple leaders
working concurrently, it takes four message delays for colliding
commands to be assigned to different slots.

[Fast Paxos](http://research.microsoft.com/pubs/64624/tr-2005-112.pdf) is a variant of Paxos that reduces
end-to-end message delays experienced by a replica to learn a chosen
value. Learning occurs in two message delays when there is no
collision, and in three message delays when there is. The price paid
is that 3f + 1 acceptors are necessary instead of 2f + 1.

In Fast Paxos, the replica sends its proposal directly to the
acceptors, bypassing the leader. Fast Paxos distinguishes fast and
classic ballots. In fast ballots, after completing the first phase,
the leader sends a p2a message without a value to the
acceptors. Concurrently, replicas send their proposals directly to
acceptors. When an acceptor receives such a p2a message, it can
choose any replica’s proposal as a value to accept. In the absence of
collisions, the acceptors end up accepting the same proposals and the
end-to-end message delay for a replica is reduced from three message
delays to two message delays.

This scheme can fail if replicas send different proposals concurrently
and the acceptors accept conflicting pvalues. In that case, a
classic ballot—which works the same as in ordinary Paxos—can be used
to recover.

As noted, Fast Paxos does not handle collision
efficiently. [Generalized Paxos](http://research.microsoft.com/pubs/64631/tr-2005-33.pdf)
improves upon Fast Paxos by
allowing independent commands to be executed in any order. Generalized
Paxos generalizes the traditional consensus problem, which chooses a
single value, to the problem of choosing monotonically increasing,
consistent values. The traditional consensus problem creates a growing
sequence of commands by assigning every command to a slot
one-by-one. This problem can be abstracted by using command
structures, or c-structs that are formed from a null element, ⊥, by
the operation of appending commands. This way, c-structs are used to
abstract command sequences. By abstracting the problem using
c-structs, it can be shown that different c-structs can belong to the
same equivalence class, that is, if a given set of commands do not
have a dependency, multiple c-structs that are equivalent to each
other can be constructed using this set of commands in different
orders. Consequently, if we use a generalized consensus algorithm that
decides on sequences of commands rather than the assignment of a
single command to a specific slot, command collisions, as they happen
in Fast Paxos, need not be resolved unless there is a direct
dependency between the concurrent commands. Different command
sequences can be accepted by different acceptors if they are in the
same equivalence class, where the ordering of specific commands do not
change the end state.

<a name="stoppable"></a>
## Stoppable Paxos

[Stoppable Paxos](http://research.microsoft.com/en-us/um/people/lamport/pubs/stoppable.pdf) implements a *stoppable replicated state* machine. The execution of a stoppable replicated
state machine can be stopped with a special command—once decided, no
more commands are executed. This variant provides an alternative
reconfiguration option: stop the current replicated state machine and
start a new one using the final application state of the previous
one. This way, the replicated state machine can be thought of as a
sequence of stoppable state machines, each running with a fixed
configuration.

<a name="mencius"></a>
## Mencius

[Mencius](http://sysnet.ucsd.edu/~yamao/pub/mencius-osdi.pdf)
tries to solve the single leader
bottleneck experienced in multi-decree Paxos. In Mencius, replicas,
acceptors, and leaders are co-located, organized in a logical ring,
and pre-assigned slot numbers to propose their commands. This way,
leaders take turns proposing commands, increasing throughput
especially when the system is CPU-bound.

Because the initial leader for a slot is known, there is no need for
phase 1 and the leader only needs to execute phase 2. If it has no
command to propose, the leader proposes a no-op command. If the
leader is faulty or slow, the other leaders can take over the slot by
executing phase 1, but they can only propose a no-op
command. Leveraging this knowledge, Mencius allows replicas to learn
about a no-op command proposed by a non-faulty leader in a single
message delay.

In normal execution without process failures, the performance of
Mencius is similar to multi-decree Paxos with a stable leader, without
the leader bottleneck. However, idle and slow leaders can diminish
overall performance. To limit the number of no-op messages generated
by leaders, Mencius uses leases between leaders, where leaders can
lease their ballots to other leaders for all slots less than a
specified slot number. This can be done voluntarily by an idle leader
giving up its ballots, or aggressively by a fast leader taking over
the ballots of a slow leader.

<a name="vertical"></a>
## Vertical Paxos

[Vertical Paxos](http://research.microsoft.com/en-us/um/people/lamport/pubs/vertical-paxos.pdf) is a Paxos variant that enables reconfiguration while the replicated state machine is active and
 deciding on commands. Vertical Paxos uses an auxiliary master to
 decide on reconfiguration operations, which determines the set of
 acceptors and the leader for every configuration.

In Vertical Paxos, the leader is set for every configuration and
different ballot numbers have different configurations. Unlike
standard “horizontal” Paxos algorithms, every configuration has
exactly one ballot number and a leader associated with it and a slot
can have more than one configuration, as shown in Figure 1(b). In
horizontal Paxos algorithms, configurations can only change when we
move horizontally from one slot to another, as in slot 3 in Figure
1(a). In Vertical Paxos, configurations change when we move
vertically, as in slot 3 in Figure 1(b), but remain the same as we
move horizontally.

<table><tr>
<td> <img src="../static/3dhpaxos.png" width="300"> <figcaption>Fig. 1(a). Horizontal Paxos</figcaption></td>
<td> <img src="../static/3dvpaxos.png" width="300"> <figcaption>Fig. 1(b). Vertical Paxos</figcaption></td>
</tr></table>

After a leader becomes active in a configuration of Vertical Paxos, it
has to communicate with the acceptors from lower-numbered ballots to
access old state. To limit the number of acceptors a leader has to
communicate with, Vertical Paxos uses two distinct quorums named read
and write quorums. When there is a configuration change, reads happen
from the read quorum in the old configuration until this state is
transferred to all live acceptors in the new configuration. Once the
state transfer is complete, the new leader informs the master and the
read and write quorums are unified.

Vertical Paxos can also be viewed as a special case of the
Primary-Backup protocol, where the leader for a given configuration
acts as a primary.

<a name="egalitarian"></a>
## Egalitarian Paxos
[Egalitarian Paxos](https://www.cs.cmu.edu/~dga/papers/epaxos-sosp2013.pdf)
(EPaxos) is another Paxos variant that
 tries to solve the single leader bottleneck by allowing all replicas
 to receive values from clients and letting them propose values with
 one message delay when there is no dependence between commands. This
 allows the system to evenly distribute the load to all replicas; in
 addition it offers better performance for geo-replicated state
 machines by enabling clients to send command requests to the replicas
 closest to them.

Unlike other Paxos variants, EPaxos orders commands dynamically and in
a decentralized fashion. EPaxos assumes that replicas, acceptors,
and leaders are co-located. In the process of proposing a command for
a slot number, a replica attaches ordering constraints to that
command, so every replica can use these constraints to independently
reach the same ordering locally.

EPaxos runs in two phases: pre-accept and accept. The pre-accept phase
establishes the ordering constraints for a command c. Upon receiving c
from a client, a replica becomes the command leader and sends a
pre-accept message including c, its dependencies depc, and a
sequence number seqc to 2f replicas. depc is the list of all instances
that contain commands that interfere with c, and seqc is a sequence
number greater than the sequence number of all interfering commands in
depc. When a replica receives a pre-accept message, it updates its
local depc and seqc attributes according to its state and replies to
the command leader with this state. The command leader might receive
replies from the quorum with depc and seqc matching in all replies—in
this case it can commit c and send a commit message to all other
replicas and reply to the client. In the other case, the command
leader might receive non-matching depc and seqc in the replies. Then,
the command leader updates its state and goes on to the accept phase.

In the accept phase, the command leader sends an accept message
including c, depc, and seqc to at least f other replicas. Upon
receiving the accept message, the replicas update their state and send
a reply to the command leader including c. When the command leader
receives at least f replies, it can send the commit message to all
replicas and reply to the client.

Following this protocol, every replica creates a local dependency
graph for every command and executes every command recursively in
accordance with this dependency graph.
