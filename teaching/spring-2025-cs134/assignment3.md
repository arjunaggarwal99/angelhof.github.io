---
layout: default
overview: true
---

<h2 class="page-heading"><a href="/teaching/distributed-systems.html">CS 134: Distributed Systems</a> -- Assignment 3: Paxos-based Key/Value Service</h2>

<h3 class="page-heading">Due date: Sunday May 18, 10pm</h3>

<hr>

<h3>Introduction</h3>

<p>
    Assignment 2 depended on a single master view server to pick the primary.
    Consequently, if the view server was not available (due to a crash or network problems),
    then the key/value service wouldn't work, even if both primary and
    backup were available. It also has the less critical defect that it
    coped with a server (primary or backup) that's briefly unavailable
    (e.g., due to a lost packet) by either blocking or declaring it dead;
    the latter is very expensive because it requires a complete key/value
    database transfer.
</p>

<h3>Overview of assignment 3</h3>
<p>
    In this assignment, you'll fix the above problems by using Paxos to manage the
    replication of a key/value store. You won't have anything
    corresponding to a master view server. Instead, a set of replicas will
    process all client requests in the same order, using Paxos to agree on
    that order. Paxos will get the agreement right even if some of the
    replicas are unavailable, or have unreliable network connections, or
    even if subsets of the replicas are isolated in their own network
    partitions. As long as Paxos can assemble a majority of replicas, it
    can process client operations. Replicas that were not in the majority
    can catch up later by asking Paxos for operations that they missed.
</p>

<p> Your system will consist of the following players: clients, kvpaxos servers,
    and Paxos peers. Clients send Put(), Append(), and Get() RPCs to key/value
    servers (called kvpaxos servers). A client can send an RPC to any of the kvpaxos
    servers, and should retry by sending to a different server if there's a
    failure. Each kvpaxos server contains a replica of the key/value database;
    handlers for client Get() and Put()/Append() RPCs; and a Paxos peer. Paxos takes the form
    of a library that is included in each kvpaxos server. A kvpaxos server talks to
    its local Paxos peer (via method calls). The different Paxos peers talk to each
    other via RPC to achieve agreement on each operation.
</p>
<p>
    Your Paxos library's interface supports an indefinite sequence of
    agreement "instances". The instances are numbered with sequence numbers. Each
    instance is either "decided" or not yet decided. A decided instance
    has a value. If an instance is decided, then all the Paxos peers that
    are aware that it is decided will agree on the same value for that
    instance. The Paxos library interface allows kvpaxos to suggest a
    value for an instance, and to find out whether an instance has been
    decided and (if so) what that instance's value is.
</p>
<p>
    Your kvpaxos servers will use Paxos to agree on the order in which
    client Put()s, Append()s, and Get()s execute. Each time a kvpaxos server receives
    a Put()/Append()/Get() RPC, it will use Paxos to cause some Paxos instance's
    value to be a description of that operation. That instance's
    sequence number determines when the operation executes relative
    to other operations. In order to find the value to be returned
    by a Get(), kvpaxos should first apply all Put()s and Append()s that are ordered
    before the Get() to its key/value database.
</p>
<p>
    You should think of kvpaxos as using Paxos to implement a "log" of
    Put/Append/Get operations. That is, each Paxos instance is a log element, and
    the order of operations in the log is the order in which all kvpaxos
    servers will apply the operations to their key/value databases. Paxos
    will ensure that the kvpaxos servers agree on this order.
</p>
<p>
    Only RPC may be used for interaction between clients and servers,
    between different servers, and between different clients. For example,
    different instances of your server are not allowed to share Go
    variables or files.
</p>
<p>
    Your Paxos-based key/value storage system will have some limitations
    that would need to be fixed in order for it to be a serious system. It
    won't cope with crashes, since it stores neither the key/value
    database nor the Paxos state on disk. It requires the set of servers to be
    fixed, so one cannot replace old servers. Finally, it is slow: many
    Paxos messages are exchanged for each client operation. All of these
    problems can be fixed.
</p>
<p>
    To help with this assignment, you should consult the Paxos-related lectures. For a wider perspective, have a look at Chubby, Paxos Made Live,
    Spanner, Zookeeper, Harp, Viewstamped Replication,
    and
    <a href="http://static.usenix.org/event/nsdi11/tech/full_papers/Bolosky.pdf">Bolosky et al.</a>
</p>

<h3>Collaboration Policy</h3>
<p>
    You must write all the code you hand in for CS134, except for code
    that we give you as part of the assignment. You are not allowed to
    look at anyone else's solution, and you are not allowed to look at
    code from previous years. You may discuss the assignments with other
    students, but you may not look at or copy each others' code. Please do
    not publish your code or make it available to future CS134 students --
    for example, please do not make your code visible on github.
</p>

<h3>Getting the Skeleton Code</h3>
<p>
<!-- <b>It is strongly recommended that you complete this assignment on the seasnet Linux servers. If you are running locally, we encourage you to use a Linux VM (e.g., using VirtualBox) and not Windows.</b><br> -->
    We have added the files needed for assignment 3 in the
    <a href="https://github.com/ucla-cs134-spring2025">same repository</a>
    as previous assignment.
    Therefore you only need to pull these new files <em>once per team</em>.
</p>


<h3>Getting started</h3>
<p>
    Assuming that you are in the <code>${TEAM_NAME}</code> directory, proceed to running the code:
</p>
<pre>
$ export GOPATH=$(pwd)
$ cd src/paxos
$ go test
Single proposer: --- FAIL: TestBasic (5.02 seconds)
        test_test.go:48: too few decided; seq=0 ndecided=0 wanted=3
Forgetting: --- FAIL: TestForget (5.03 seconds)
        test_test.go:48: too few decided; seq=0 ndecided=0 wanted=6
...
</pre>
<p>
    For now, ignore the huge number of "has wrong number of ins" and "type Paxos
    has no exported methods" errors.
</p>

<h3>Part A: Paxos</h3>

<p>
    First you'll implement a Paxos library.
    <tt>paxos.go</tt> contains descriptions of the methods you must
    implement. When you're done, you should pass all the tests in the
    <tt>paxos</tt> directory (after ignoring Go's many complaints):
</p>
<pre>
$ go test
Test: Single proposer ...
... Passed
Test: Many proposers, same value ...
... Passed
Test: Many proposers, different values ...
... Passed
Test: Out-of-order instances ...
... Passed
Test: Deaf proposer ...
... Passed
Test: Forgetting ...
... Passed
Test: Lots of forgetting ...
... Passed
Test: Paxos frees forgotten instance memory ...
... Passed
Test: Many instances ...
... Passed
Test: Minority proposal ignored ...
... Passed
Test: Many instances, unreliable RPC ...
... Passed
Test: No decision if partitioned ...
... Passed
Test: Decision in majority partition ...
... Passed
Test: All agree after full heal ...
... Passed
Test: One peer switches partitions ...
... Passed
Test: One peer switches partitions, unreliable ...
... Passed
Test: Many requests, changing partitions ...
... Passed
PASS
ok      paxos   59.523s
</pre>

<p>
    Your implementation must support this interface:
</p>
<pre>
px = paxos.Make(peers []string, me int)
px.Start(seq int, v interface{}) // start agreement on new instance
px.Status(seq int) (fate Fate, v interface{}) // get info about an instance
px.Done(seq int) // ok to forget all instances <= seq
px.Max() int // highest instance seq known, or -1
px.Min() int // instances before this have been forgotten
</pre>

<p>
    An application calls Make(peers,me) to create a Paxos peer. The peers
    argument contains the ports of all the peers (including this one), and
    the me argument is the index of this peer in the peers array.
    Start(seq,v) asks Paxos to start agreement on instance seq, with
    proposed value v; Start() should return immediately, without waiting
    for agreement to complete. The application calls Status(seq) to find
    out whether the Paxos peer thinks the instance has reached agreement,
    and if so what the agreed value is.
    Status() should consult the local Paxos peer's state and return
    immediately; it should not communicate with other peers.
    The application may call Status()
    for old instances (but see the discussion of Done() below).
</p>
<p>
    Your implementation should be able to make progress on agreement for
    multiple instances at the same time. That is, if application peers
    call Start() with different sequence numbers at about the same time,
    your implementation should run the Paxos protocol concurrently for all
    of them. You should <b>not</b> wait for agreement to complete for
    instance i before starting the protocol for instance i+1. Each instance
    should have its own separate execution of the Paxos protocol.

</p>
<p>
    A long-running Paxos-based server must forget about
    instances that are no longer needed, and free the memory
    storing information about those instances.
    An instance is needed if the
    application still wants to be able to call Status() for that instance,
    or if another Paxos peer may not yet have reached agreement on that
    instance. Your Paxos should implement freeing of instances in the
    following way. When a particular peer application will no longer need
    to call Status() for any instance <= x, it should call Done(x). That Paxos peer can't yet discard the instances, since some other Paxos peer might not yet have agreed to the instance. So each Paxos peer should tell each other peer the highest Done argument supplied by its local application. Each Paxos peer will then have a Done value from each other peer. It should find the minimum, and discard all instances with sequence numbers <=that minimum. The Min() method returns this minimum sequence number plus one. </p> <p>
        It's OK for your Paxos to piggyback the Done value in the agreement
        protocol packets; that is, it's OK for peer P1 to only learn P2's
        latest Done value the next time that P2 sends an agreement message to
        P1. If Start() is called with a sequence number less than Min(),
        the Start() call should be ignored. If Status() is called with a
        sequence number less than Min(), Status() should return <tt>Forgotten</tt>.
</p>
<p>
    Here is the Paxos pseudo-code (for a single instance):
</p>
<pre>

proposer(v):
while not decided:
    choose n, unique and higher than any n seen so far
    send prepare(n) to all servers including self
    if prepare_ok(n, n_a, v_a) from majority:
    v' = v_a with highest n_a; choose own v otherwise
    send accept(n, v') to all
    if accept_ok(n) from majority:
        send decided(v') to all

acceptor's state:
n_p (highest prepare seen)
n_a, v_a (highest accept seen)

acceptor's prepare(n) handler:
if n > n_p
    n_p = n
    reply prepare_ok(n, n_a, v_a)
else
    reply prepare_reject

acceptor's accept(n, v) handler:
if n >= n_p
    n_p = n
    n_a = n
    v_a = v
    reply accept_ok(n)
else
    reply accept_reject

</pre>

<p>
    Here's a reasonable plan of attack:
</p>
<ol>

    <li> Add elements to the <tt>Paxos</tt> struct in <tt>paxos.go</tt>
        to hold the state you'll need, according to the lecture pseudo-code.
        You'll need to define a struct to hold information about each
        agreement instance.
    </li>
    <li> Define RPC argument/reply type(s) for Paxos protocol messages,
        based on the lecture pseudo-code. The RPCs must include the sequence
        number for the agreement instance to which they refer. Remember
        the field names in the RPC structures must start with capital letters.</li>

    <li> Write a proposer function that drives the Paxos protocol for
        an instance, and RPC handlers that implement acceptors. Start a
        proposer function in its own thread for each instance, as needed
        (e.g. in Start()).
    </li>
    <li> At this point you should be able to pass the first few tests.
    </li>
    <li> Now implement forgetting.
    </li>
</ol>

<p>
    Hint: more than one Paxos instance may be executing at a given time,
    and they may be Start()ed and/or decided out of order (e.g. seq 10 may
    be decided before seq 5).
</p>
<p>
    Hint: in order to pass tests assuming unreliable network, your paxos
    should call the local acceptor through a function call rather than RPC.
</p>
<p>
    Hint: remember that multiple application peers may call Start() on the
    same instance, perhaps with different proposed values. An application
    may even call Start() for an instance that has already been decided.
</p>
<p>
    Hint: think about how your paxos will forget (discard) information about
    old instances before you start writing code. Each Paxos peer will need to
    store instance information in some data structure that allows
    individual instance records to be deleted (so that the Go garbage
    collector can free / re-use the memory).
</p>
<p>
    Hint: you do not need to write code to handle the situation where
    a Paxos peer needs to re-start after a crash. If one of your
    Paxos peers crashes, it will never be re-started.
</p>
<p>
    Hint: have each Paxos peer start a thread per un-decided instance
    whose job is to eventually drive the instance to agreement, by
    acting as a proposer.
</p>
<p>
    Hint: a single Paxos peer may be acting simultaneously as acceptor and
    proposer for the same instance. Keep these two activities as separate
    as possible.
</p>
<p>
    Hint: a proposer needs a way to choose a higher proposal number than
    any seen so far. This is a reasonable exception to the rule that
    proposer and acceptor should be separate. It may also be useful for the
    propose RPC handler to return the highest known proposal number if it
    rejects an RPC, to help the caller pick a higher one next time.
    The <tt>px.me</tt> value will be different in each Paxos peer,
    so you can use <tt>px.me</tt> to help ensure that proposal numbers
    are unique.
</p>
<p>
    Hint: figure out the minimum number of messages Paxos should use when
    reaching agreement in non-failure cases and make your implementation
    use that minimum.
</p>
<p>
    Hint: the tester calls Kill() when it wants your Paxos to shut down;
    Kill() sets px.dead. You should call <tt>px.isdead()</tt> in any loops you have
    that might run for a while, and break out of the loop if <tt>px.isdead()</tt> is
    true. It's particularly important to do this any in any long-running
    threads you create.
</p>
<h3>Part B: Paxos-based Key/Value Server</h3>
<p>
    Now you'll build kvpaxos, a fault-tolerant key/value storage system.
    You'll modify <tt>kvpaxos/client.go</tt>,
    <tt>kvpaxos/common.go</tt>, and <tt>kvpaxos/server.go</tt>.
</p>
<p>
    Your kvpaxos replicas should stay identical; the only exception
    is that some replicas may lag others if they are not
    reachable. If a replica isn't reachable for a while, but then starts
    being reachable, it should eventually catch up (learn about operations
    that it missed).
</p>
<p>
    Your kvpaxos client code should try different replicas it knows about
    until one responds. A kvpaxos
    replica that is part of a majority of replicas that can all reach each
    other should be able to serve client requests.
</p>
<p> Your storage system must provide sequential consistency to applications that
    use its client interface. That is, completed application calls to the
    Clerk.Get(), Clerk.Put(), and Clerk.Append() methods in
    <tt>kvpaxos/client.go</tt> must appear to have affected all replicas in the same
    order and have at-most-once semantics. A Clerk.Get() should see the value
    written by the most recent Clerk.Put() or Clerk.Append() (in that order) to the same key. One
    consequence of this is that you must ensure that each application call to
    Clerk.Put() or Clerk.Append()
    must appear in that order just once (i.e., write the key/value
    database just once), even though internally your <tt>client.go</tt> may have to
    send RPCs multiple times until it finds a kvpaxos server
    replica that replies.
</p>
<p>
    Here's a reasonable plan:
</p>
<ol>

    <li> Fill in the Op struct in server.go with the "value" information
        that kvpaxos will use Paxos to agree on, for each client request.
        Op field names must start with capital letters. You should use
        Op structs as the agreed-on values -- for example, you should pass
        Op structs to Paxos Start(). Go's RPC can marshall/unmarshall
        Op structs; the call to gob.Register() in StartServer() teaches
        it how.
    </li>
    <li> Implement the PutAppend() handler in server.go. It should enter a Put
        or Append
        Op in the Paxos log (i.e., use Paxos to allocate a Paxos instance,
        whose value includes the key and value (so that other kvpaxoses know
        about the Put() or Append())).
        An Append Paxos log entry should contain the Append's arguments,
        but not the resulting value, since the result might be large.
    </li>
    <li> Implement a Get() handler. It should enter a Get Op in the Paxos
        log, and then "interpret" the the log before that point to make sure
        its key/value database reflects all recent Put()s.
    </li>
    <li> Add code to cope with duplicate client requests, including
        situations where the client sends a request to one kvpaxos replica,
        times out waiting for a reply, and re-sends the request to a different
        replica. The client request should execute just once. Please make sure
        that your scheme for duplicate detection frees server memory quickly, for
        example by having the client tell the servers which RPCs it has heard a
        reply for. It's OK to piggyback this information on the next client
        request.
    </li>
</ol>

<p>
    Hint: your server should try to assign the next available Paxos
    instance (sequence number) to each incoming client RPC. However, some
    other kvpaxos replica may also be trying to use that instance for a
    different client's operation. So the kvpaxos server has to be prepared
    to try different instances.
</p>
<p>
    Hint: your kvpaxos servers should not directly communicate; they
    should only interact with each other through the Paxos log.
</p>
<p>
    Hint: as in assignment 2, you will need to uniquely identify client
    operations to ensure that they execute just once.
    Also as in assignment 2, you can assume that each clerk has only one
    outstanding Put, Get, or Append.
</p>
<p>
    Hint: a kvpaxos server should not complete a Get() RPC if it is not
    part of a majority (so that it does not serve stale data). This means
    that each Get() (as well as each Put() and Append()) must involve Paxos
    agreement.
</p>
<p>
    Hint: don't forget to call the Paxos Done() method when a kvpaxos
    has processed an instance and will no longer need it or any
    previous instance.
</p>

<p>
    Hint: your code will need to wait for Paxos instances to complete
    agreement. The only way to do this is to periodically call Status(),
    sleeping between calls. How long to sleep? A good plan is to check
    quickly at first, and then more slowly:
</p>
<pre>
    to := 10 * time.Millisecond
    for {
    status, _ := kv.px.Status(seq)
    if status == paxos.Decided{
        ...
        return
    }
    time.Sleep(to)
    if to < 10 * time.Second {
        to *= 2
    }
    }
    </pre>

<p>
    Hint: if one of your kvpaxos servers falls behind (i.e. did not
    participate in the agreement for some instance), it will later need to
    find out what (if anything) was agree to. A reasonable way to to this
    is to call Start(), which will either discover the previously
    agreed-to value, or cause agreement to happen. Think about what value
    would be reasonable to pass to Start() in this situation.
</p>
<p>
    Hint: When the test fails, check for gob error (e.g. "rpc: writing
    response: gob: type not registered for interface ...") in the log because go
    doesn't consider the error fatal, although it is fatal for the assignment.
</p>
<h3>Assignment Submission</h3>

<p>
    To submit the assignment, please push your final code into your team's repository.
    Then use Gradescope to turn in your assignment from GitHub. Remember to add your teammate
    to your Gradescope submission.
</p>
<p>
You will receive full credit if your code passes (provided there is no plagiarism) the <tt>test_test.go</tt> tests. You can submit the assignment 2 days late at the max(provided both of the team members have 2 late days remaining). To use a late day, you can just use the same method for submission on Gradescope. We have a late due date mentioned on the assignment that will handle it accordingly. Each used late day pushes the due date back by exactly 24 hours.
<!-- In the event that no late days are used, we will evaluate the code in your <strong>last</strong> submission prior to the deadline. -->
</p>

<hr>

<address>
    Please post questions on <a href="https://piazza.com/ucla/spring2025/cs134">Piazza</a>.
</address>