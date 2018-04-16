# SWIM (Scalable Weakly-Consistent Infection-Style Process Group Membership) Protocol

#### http://www.cs.cornell.edu/projects/quicksilver/public_pdfs/SWIM.pdf

## 2 Previous work
- Previous implementations rely either on:
  - Sending periodic heartbeats to every other node in the cluster
    - Quadratic in number of messages as nodes are added to cluster
  - Sending periodic heartbeats to a subset of nodes
    - Requires maintaining an overlay graph
  - Key properties of a distributed failure detector protocol
    - **Strong completeness**: crash-failure of any group-member is detected by all non-faulty members
    - **Speed of failure detection**: the time interval between a member failure and its detection by *some* non-faulty member
    - **Accuracy**: the rate of false positives of failure detection
## 3 The Basic SWIM Approach
- 2 major components
  - Failure detector component: detect failures of members
  - Dissemination component: disseminate information about members that have joined or left the group, or failed
### 3.1 SWIM Failure Detector
- 2 parameters
  - *T'*: protocol period (not dependent on synchronized clocks)
  - *k*: size of failure subgroups
- Every *T'* time units, *M<sub>i</sub>* chooses a random member, *M<sub>j</sub>*, and sends a *ping* message to it
  - Then waits for an *ack* message
  - If *ack* not received within timeout (greater than RTT, less than *T'*)
    - Send *ping-req(*M<sub>j</sub>*)* to *k* random members
    - Each process then sends a *ping* to *M<sub>j</sub>*, and forwards response (if any) to *M<sub>i</sub>*
  - At end of each protocol period, *M<sub>i</sub>* checks to see if it has received an *ack* from *M<sub>j</sub>*
    - If it hasn't, it marks it as faulty in its local membership list, and hands it off to Dissemination Component
- Protocol period must be at least 3xRTT
  - Message timeout can be slightly larger than RTT, with protocol period much larger than RTT
- Each message is tagged with a unique sequence number
- Messages are of constant size, indepdendent of group size
### 3.2 Dissemination Component and Dynamic Membership
- When discovering failure, multicast to rest of group
  - Each member deletes the faulty member from local membership list
- Joining/leaving group can be similarly multicast
  - Joining members must have some way of contacting existing members (e.g. DNS/static IP, IP multicast, network broadcast)
## 4 A More Robust and Efficient SWIM
- IP multicast is only best-effort
- Combine membership updates with *ping* and *ack* messages
- Protocol is subject to non-network disturbances (e.g. network buffers, overloaded host)
  - Processes can be marked *suspected* before being declared *faulty*
  - Process can respond before timeout and be declared healthy
- Every failure will eventually be detected by every non-faulty member, but there is no deterministic time bound
### 4.1 Infection-Style Dissemination Component
- Instead of relying on hardware/software multicast, build it into *ping*, *ping-req*, and *ack* messages
- Each member maintains a list of recent membership updates, with a local count of each buffer element
  - Local count specifies number of times the message has been gossipped
- If buffer limit has been reached, prefer least-gossipped messages
  - Ensures buffer is not overwhelmed by member churn
- Maintain 2 lists of group members
  - Members not yet declared failed in the group
  - Members that have failed recently
  - Equal number from each list are chosen to attach to a message (but could be adapted)
### 4.2 Suspicion Mechanism: Reducing the Frequency of False Positives
- If a member failure is detected by a single member, it is forced out of the group
  - Leads to high rate of false positives
- Run *Suspicion* subprotocol when failure is first detected
  - If at end of normal *SWIM* failure detection protocol *M<sub>j</sub>* is detected as failed
    - Marked as *suspected* in local membership list
    - `{Mj suspected by Mi}` message disseminated through group through Dissemination Component
    - Other processes also mark *M<sub>j</sub>* as suspected in local membership lists
    - Suspected processes stay in membership lists, treated similarly to non-faulty members
- If *M<sub>l</sub>* successfully *ping*s *M<sub>j</sub>*, it disseminates a `{Ml knows Mj is alive}` messages
  - An *Alive* message un-marks the process as *suspected*
  - Similarly, if *M<sub>j</sub>* sees that it is suspected, it can disseminate its own *Alive* message
- Suspected members expire after timeout
- If after timeout, *M<sub>h</sub>* hasn't seen an *Alive* message, it sends a `{Mh declares Mj as faulty}` message
  - Overrides previous *Suspect* and *Alive* messages
- *Alive* overrides *Suspect*, *Confirm* overrides *Alive* and *Suspect*
- Unique *incarnation number*s are needed to distinguish between multiple message types for a single process (over group lifetime)
  - Initalized to 0 on member startup
  - Incremented only by local process when information about itself being suspected is seen in Dissemination Component
  - Sends *Alive* message with identifier and incremented incarnation number
### 4.3 Round-Robin Probe Target Selection: Providing Time-Bounded Strong Completeness
- The basic SWIM failure detector protocol detects failures in an average constant number of protocol periods
- In the pathological case, there may be large delay in first detection of a failed process
  - In the extreme case, the delay is unbounded
- Instead, select *ping* targets sequentially from list
  - New members are inserted randomly in list
  - When list has been traversed, list is shuffled
  - Sending *ping* messages is at most (`2n - 1`, `n` is number of members) protocol periods apart
