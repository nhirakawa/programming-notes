# Time, Clocks, and the Ordering of Events in a Distributed System

#### http://amturing.acm.org/p558-lamport.pdf

## Introduction
- A distributed system consists of a collection of distinct processes, spatially separated, and which communicate by exchanging messages.
  - A system is distributed if message transmission delay is non-negligible compared to the time between events in a single process
- In a distributed system, it is sometimes impossible to say that an event occurred before another
  - "Happened before" is only a partial ordering
## The Partial Ordering
- Clocks are not perfectly accurate, and do not keep precise physical time
  - Need to define "happened before" relation without using physical clocks
- A system is composed of processes
  - Each process has a sequence of events
- The events of a process form a sequence
  - *a* occurs before *b* if *a* happens before *b*
  - A single process is a set of events with an a priori total ordering
- **Definition**: The relation -> on the set of events of a system is the smallest relation satisfying the following three conditions
  1. If *a* and *b* are events in the same process, and *a* comes before *b*, then *a* -> *b*
  1. If *a* is the sending of a message by one process, and *b* is the recept of the same message by another process, then *a* -> *b*
  1. If *a* -> *b* and *b* -> *c*, then *a* -> *c*
- 2 distinct events *a* and *b* are concurrent if *a* !-> *b* and *b* !-> *a*
  - Assume *a* !-> *a*
## Logical clocks
- Define *C<sub>i</sub>* for *P<sub>i</sub>* which assigns a number *C<sub>i</sub>{a}* to any event *a* in the process
  - Entire system of clocks is represented by function `C` which assigns to any event *b* the number *C{b}* where *C{b}=C<sub>j</sub>{b}* if *b* is an event in process *P<sub>j<sub>*
- **Clock Condition**: For any events *a*, *b*
  - if *a* -> *b*, then *C{a}<C{b}* 
  - Cannot expect converse condition to hold
  - Satisfied if 2 conditions hold
     1. If *a* and *b* are events in process *P<sub>i</sub>*, and *a* comes before *b*, then *C<sub>i</sub>{a}<C<sub>i</sub>*
     1. If *a* is the sending of a message by process *P<sub>i</sub>* and b is the receipt of that message by process *P<sub>j</sub>*, then *C<sub>i</sub>{a}<C<sub>j</sub>{b}*
- Now introduce clocks that satisfy the **Clock Condition**
  - *P<sub>i</sub>*'s clock is represented by register *C<sub>i</sub>*
  - *C<sub>i</sub>{a}* is the value contained by *C<sub>i</sub>* during event *a*
    - *C<sub>i</sub>* changes between events
- **Implementation Rule**: Each process *P<sub>i</sub>* increments *C<sub>i</sub>* between any two successive events
  - To satisfy **Clock Condition** 1
- **Implementation Rule**:
  - If event *a* is the sending of a message *m* by *P<sub>i</sub>*, then the message *m* contains a timestamp *T<sub>m</sub> = C<sub>i</sub>{a}*
  - Upon receiving message *m*, process *P<sub>j</sub>* sets *C<sub>j</sub>* greater than or equal to its present value and greater than *T<sub>m</sub>*
## Ordering the Events Totally
- To place a total ordering on the set of all system events, simply use the times at which they occur
  - Break ties with an aribtary total ordering *<*
  - Define *=>* relation
    - If *a* is a is an event in *P<sub>i</sub>* and *b* is an event in *P<sub>j</sub>*, then *a => b* iff
      - *C<sub>i</sub>{a} < C<sub>j</sub>{b}*, or
      - *C<sub>i</sub>{a} = C<sub>j</sub>{b}* and *P<sub>i</sub> < P<sub>j</sub>* 
  - Depends on system of clocks *C<sub>i</sub>* and is not unique
- Use total ordering of events to solve mutual exclusion problem
  - Must satisfy 3 conditions
    1. A process which has been granted the resource must releast it before it can be granted to another process
    1. Different requests for the resource must be gratned in the order in which they are made
    1. If every process which is granted the resource eventually releases it, then every request is eventually granted
  - Assume that resource is initially granted to exactly one process
  - Assumptions
    - Assume messages are received in the same order that they are sent
    - Every message is eventually received
    - Every process can send messages directly to every other process
  - Each process maintains a request queue
  - Five rules:
    1. To reqeust the resource, P<sub>i</sub> send message T<sub>m</sub>:P<sub>i</sub> to every other process and puts the message on its queue; T<sub>m</sub> is the timestamp of the message
    1. When process P<sub>j</sub> receives the message, it places it on its queue and sends a timestamped acknowledgement
    1. To relase resource, process P<sub>i</sub> removes messages from its queue and sends a timestamped message to every other process.
    1. When process P<sub>j</sub> receives a message, it removes any messages from its queue
    1. Process P<sub>i</sub> is granted resource when 2 conditions are met
      1. There is a T<sub>m</sub>:P<sub>i</sub> message in its queue which is ordered before any other request in the queue (ordered by *=>*)
      1. P<sub>i</sub> has received a message from every other process timestamped later than T<sub>m</sub>
## Physical Clocks
- Let *C<sub>i</sub>(t)* be the reading of clock *C<sub>i</sub>* at physical time *t*
  - Assume clocks run continuously (i.e. non-discretely)
  - Assume *C<sub>i</sub>(t)* is continuous and differentiable everywhere except for where the clock is reset
  - *dC<sub>i</sub>(t)/dt* represents tha rate at which the clock is running at *t*
- To be a true physical clock, it must run at *dC<sub>i</sub>(t)/dt ~ 1*
- Assume that the following condition is satisfied:
  - (PC1) There exists a constant *k << 1* such that for all *i: |dC<sub>i</sub>(t)/dt - 1| < k*
  - For typical crystal controlled clocks, *k <= 10^-6*
- Not enough for clocks to run at the same rate
- Must also be synchtonized, so that:
  - (PC2) For all *i*, *j*: *|C<sub>i</sub>(t) - C<sub>j</sub>(t)| < e*
- Since two different clocks will never run at the same rate, they will only drift further apart
  - Must devise an algorithm to make sure (PC2) always holds
- Let *u* be a number such that if event *a* occurs at physical time *t*, and event *b* in another process satisfies *a -> b*, then *b* occurs later than physical time *t + u*
  - *u* is the shortest possible transmission time
  - Can be no shorter than speed of light multiplied by physical distance, but may be much larger
- Must make sure for any *i*, *j*, *y*: *C<sub>i</sub>(t+u) - C<sub>j</sub>(t) > 0*
- Specialized rules IR1 and IR2
  - IR1' - For each *i*, if *Pi* does not receive a message at physical time *t*, then *Ci* is differentiable at *t* and *dCi(t)/dt > 0*
  - IR2'
    - (a) If *Pi* sends a message *m* at physical time *t*, then *m* contains a timestamp *Tm = Ci(t)*
    - (b) Upon receiving a message *m* at physical time *t'*, process *Pj* sets *Cj(t')* to maximum *(Cj(t'-0), Tm + um)*
- A process only needs to know its own clock and the timestamp of the message
  - Assume each message occurs at a precise instant of physical time
  - Assum different events in the same process occur at different times
