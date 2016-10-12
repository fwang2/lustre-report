# Adaptive timeout

This feature is enabled by default in 1.8. We describe the need or use case for
such feature, and implementation details below.

## Why this feature?

Both client and server in Lustre use timeout: for example, clients use it to
trigger the recovery when server goes down, and servers use the timeout to
determine if the client is alive to respond a lock revocation request.
If server didn't get back response from client during lock callback, a client
gets evicted, as shown in the following log message.
 
    LustreError: ldlm_lockd.c:190:waiting_locks_callback()) 
    ### lock callback timer expired: evicting client ...

The problem is that this timeout value is fixed in previous Lustre, with
`obd_timeout` served as the base for calculation, tuned with
`/proc/sys/lustre/timeout` Large scale cluster tends to exhibit large
variance on the response time depending on the overall load of the system. The
downside of setting it short is obvious as you see from above example. Setting
it too high is also not a viable option in a large cluster as it renders the
recovery period exceedingly long.


## Mechanisms

* Client estimates RPC timeout is based on round trip RPC time as well
  as RPC serving time. The later is embedded in service reply RPC.
  Conversely, the estimated client RPC timeout information is sent along
  client RPC request.
  
* If server can't serve the client RPC request within the time frame of
  client RPC timeout, server will send so-called \texttt{keep alive} reply to
  help client adjust its timeout estimation.
  
* Server maintains current worst-case RPC service time: measured from the
  time LNET delivers the RPC request to the time server posts the normal RPC
  reply, over a period of $N$ requests, for example.
  
* Server reply (both normal RPC and keep-alive RPC) will include this
  worst-case RPC service time plus the elapsed time for the RPC request.
   
