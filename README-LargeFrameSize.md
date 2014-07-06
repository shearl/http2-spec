
Large Frame Sizes Proposal
==========================

Background
----------
The HTTP2 protocol has a requirement to be able to transport large headers, 
that exceed the payload size of a single frame at the current 16KB maximim size.  

To address this requirement, the current draft (13) includes the 
CONTINUATION frames, 0 or more of which may be sent after a HEADERS or 
PUSH_PROMISE frame to contain the large headers.  There has been significant 
criticism of the CONTINUATION design, including:

 * The total length of a HEADERS+CONTINUATION* sequence is not known until 
   the last frame in the sequence is processed.  A receiver that wishes to 
   reject streams headers larger than a specific limit may have to process 
   many frames and hold the results in memory before it discovers the header
   is too large.

 * The size of header that an endpoint is prepared to receive is not known
   in advance. The only way a sender can know if a header too large is
   by attempting to send it and receiving an error in response. Error handling
   of headers may be difficult for an endpoint to handle efficiently and 
   can result in the closure of the entire connection.

 * The END_STREAM flag is not present on the CONTINUATION frame, thus it is 
   possible for a stream to send CONTINUATION frames after a HEADERS frame 
   that has the END_STREAM flag set.  This is confusing and increases the 
   complexity of the state machine required to process streams.  It is 
   highly desirable that a set END_STREAM flag truly indicates the last
   non control frame of a stream.

 * There is a significant discontinuity in the code path required to 
   process headers.  Headers up to an indeterminant size (roughly 20-something KB)
   can be handled in a single frame. Headers that exceed this size must
   be handled in multiple frames of different types with different frame flags
   and stream control logic. Because the vast majority of headers sent (>99.99%) 
   are below this indeterminant size, implementations will have a code path
   that is seldom executed and probably insufficiently tested. This invites
   poor and/or partial and/or incorrect implementation.

 * Because of the HPACK compression algorithm, a sequence of
   HEADERS+CONTINUATIONS frames may not be interleaved with any other frame.  
   This effectively makes the sequence a single large frame.  Because of the simplicity
   of description and implementation it is proposed that it would be
   far simpler to meet the requirement of large headers by supporting
   large frames.


This proposal has been prepared as it is possible to meet the requirements
of CONTINUATIONS without the complications and criticisms above.

> This propossal addresses the issue of sending/receivign large HTTP 
> headers without giving endpoints and intermediaries unlimited 
> resource commitments nor unknown limits


Additional Frame Size Issues Addressed
--------------------------------------
The current draft (13) has maximume frame size of 16KB, which is an
arbitrary value that has been selected on the basis of experience to
provide a reasonable compromise between the efficiency of transmitting
data vs the quality of service for multiplexed channels.  

Whilst this educated guess may be near optimal for todays networks 
and traffic, it is entirely possible that some current and/or future 
networks may require a different value to achieve an optimal balance.  
There have already been proposals [1] put to the WG to reduce the 
frame size for slower networks, as well as discussion that 
high capacity, low latency networks can achieve satisfactory multiplexing 
quality of service with large frame sizes. 

> This proposal addresses the issue of having a fixed frame size for 
> tuning of multiplexing performance based on current/future experience.

It has also been noted that 16KB is near the middle of the peak of the
current HTTP Object size histogram [2], so that a small change in the frame 
size may have a significant impact on the number of HTTP messages that 
can be sent in a single frame, without significant impacts on QoS. The
HTTP Object size histogram has changed signifcantly over time and is 
expected to continue to do so.

> This proposal addresses the issue of tuning the frame size based on
> experience of actual payload sizes.


There have also been issues raised that a 16KB frame size does not allow
efficient data transfer [3] even when the end points are awway that only
a single stream is likely to be required for the imminent future, or 
that a particular stream is of high priority.  

> This proposal addresse the issue of tuning the frame size for transport
> efficiency for specific streams in specific situations.



Large Frame Header Proposal
---------------------------
This proposal is to alter the core http2 protocol so that addresses 
the issues identified above by supporting a large frame size that
will have variable size limits applied.

This proposals increases the length field in the frame header to 
31 bits, to match the maximum flow control window size.  However,
implementations will not be able to use the full frame size without
explicit concent from peers.


Frame Size Settings
-------------------
Two settings parameters are proposed: SETTINGS_HEADER_FRAME_SIZE
for the maximum header size and SETTINGS_FRAME_SIZE for all other frames.

The SETTINGS_HEADER_FRAME_SIZE parameter supports the current behaviour
where large headers can be sent without changing the frame size allowed 
for other frame types. ie A large header size limit can be set without
affecting the multiplexing efficiency of DATA frames.

The SETTINGS_FRAME_SIZE applies to all other frames including DATA frames
and any other frame that may be defined by an extension.  The use of
this parameter is intended to tune/optimise the connection for the 
general case of multiple streams over the specific connection.


Frame Size Updates
------------------
To handle the issue of efficiently sending large data when an end point
is prepared to risk multiplexing efficiency, this proposal allows
a Max Frame Size to be applied to a specific stream as an optional field
in a WINDOW_UPDATE frame.

By including a variable frame size in the flow control mechanism this 
proposal allows the decision to increase the frame size to be deferred until
more knowledge about the specific situation are known and limited to the
stream that will benefit from the increased sized.

Consider the example of a server that has commenced sending a large content
to a client.  The server may initially send 4 x 16KB frames to consume the 
default stream flow control window, at which time it must wait for the client
to send a WINDOW_UPDATE frame before continuing. When generating the 
WINDOW_UPDATE frame, the client may have knowledge of:
 * The content-length header so it knows that the amount of data expected 
   is large are just slightly larger than a single frame.
 * The content-type header so it knows if the content has high priority
   in rendering the current page, or if the content is likely to include
   references to other resources.  
 * How many other streams are current open and/or reserved 
 * How many other requests are pending from the content received so far.  
 * An appriximate rough measure of the network latency and throughput 
   derived from the timing of the receipt of the first few data frames.

The client can use this knowledge to make an informed decision as to the 
risk multiplexing QoS by increasing the frame size.   It can make several 
choices:

 * No change. It can decide that it is too hard to consider or that there
   are too many other streams, or that the content is video that needs to
   be received slowly.  In any of these cases it can just not adjust the
   frame size and the protocol continues as it currently does.  

 * Large frame. If the stream is the only expected stream or of
   sufficiently high priortiy, then the window and frame size can be set
   to allow as much of the remaining data as possible to be sent in a
   single frame.

 * Medium frame. The client can momentarily trade some QoS by an
   estimated duration by increasing the frame size to something >16KB 
   and < content-length

 * Sufficient frame.  If the remaining content is only a small increment
   over the current SETTING_FRAME_SIZE, the Max Frame Size can be increased
   to receive the remaining content in a single frame without any 
   significant QoS impact.


Minimal Compliance
-------------------
A minimally compliant implementation MUST handle the SETTING_FRAME_SIZE
and SETTINGS_HEADER_SIZE and ensure that no frame sent exceeds the
applicable limit.   However no implementation is required to send frames 
at or near these limits when set above the default 16KB. 

There is no requirement for an implementation to send or to handle the 
Max Frame Size in a WINDOW_UPDATE and it is allowable for it to be ignored
if received.


Anticipated Criticism
--------------------

> It is too late in the process to change the framing layer and to do
> so after so much discussion is an implicit fail of the WG

To not consider issues and proposal brought to the WG would be a fail of 
the process.   This proposal is based on all the hard work to date done
by the WG and contributors to identify issues and test solutions.

> These issues can be handled in extensions.

Optimising data transfers for large content could possibly be done in an
extension, however:
  * It is not yet clear if extensions will be a viable way to enhance the
    http2 protocol. There are significant hurdle to overcome to deploy
    extensions.
  * Many of the issues are aimed at complexity and tuning of the core
    protocol, and these cannot be addressed in an extension.
  * It is asymmetric to support large headers with one mechanism and large
    data with another.

> The proposed header costs 2 extra bytes per frame

There is a small data cost to adopt this proposal, however this is 
mitigated as:
  * The proposal may be able to reduce the number of frames needed
    for some content, thus saving 8 bytes.  Whilst not likely to be
    a 25% frame saving required to break even, it will still reduce
    cost to below 2 bytes.
  * There are options to have variable length headers or optional
    extended headers that will preserve the semantics of this proposal
    and keep an 8 byte header for small frames.  If the 2 byte cost
    is considered prohibitive, then these alterations can be considered.

> The header is 10 bytes long and not 32bit word aligned.

Frames sent after arbitrary data will not be word aligned anyway.

> 31 bits is also an arbitrary length

It is true that a 31 bit large frame length is also an arbitrary 
limit to the size of a frame. However, it is beleived that 31 bits is
sufficiently large to efficiently handle almost all concievable present
and future use cases.   It would be possible to implement an unlimited
size length field, but it is felt that this complexity is not worthwhile
given the low probability of it being required.



[1] http://lists.w3.org/Archives/Public/ietf-http-wg/2013AprJun/0926.html
[2] http://httparchive.org/interesting.php
[3] http://lists.w3.org/Archives/Public/ietf-http-wg/2014AprJun/1664.html
