# TCP Receive Buffer

## How to check receive buffer?
```
$ ss -mt
State     Recv-Q      Send-Q                       Local Address:Port                        Peer Address:Port             Process                                                   
...
ESTAB     0           0                            192.168.1.102:60200                      140.82.113.26:https            
	 skmem:(r0,rb131072,t0,tb87040,f0,w0,o0,bl0,d1)         
...
```

The value of **rb**(receive buffer) is the size if socket receive buffer.


## Socket buffer explanation

### Background

From `man 7 tcp`:

```
The  maximum  sizes  for  socket  buffers  declared  via  the  SO_SNDBUF  and  SO_RCVBUF  mechanisms  are  limited  by  the values in the /proc/sys/net/core/rmem_max and
       /proc/sys/net/core/wmem_max files.  Note that TCP actually allocates twice the size of the buffer requested in the setsockopt(2) call, and so a succeeding  getsockopt(2)
       call  will  not  return  the same size of buffer as requested in the setsockopt(2) call.  TCP uses the extra space for administrative purposes and internal kernel struc‐
       tures, and the /proc file values reflect the larger sizes compared to the actual TCP windows.  On individual connections, the socket buffer size must be set prior to the
       listen(2) or connect(2) calls in order to have it take effect.  See socket(7) for more information.
```

```
       tcp_rmem (since Linux 2.4)
              This is a vector of 3 integers: [min, default, max].  These parameters are used by TCP to regulate receive buffer sizes.  TCP dynamically adjusts the size of  the
              receive buffer from the defaults listed below, in the range of these values, depending on memory available in the system.

              min    minimum  size of the receive buffer used by each TCP socket.  The default value is the system page size.  (On Linux 2.4, the default value is 4 kB, lowered
                     to PAGE_SIZE bytes in low-memory systems.)  This value is used to ensure that in memory pressure mode, allocations below  this  size  will  still  succeed.
                     This is not used to bound the size of the receive buffer declared using SO_RCVBUF on a socket.

              default
                     the  default  size of the receive buffer for a TCP socket.  This value overwrites the initial default buffer size from the generic global net.core.rmem_de‐
                     fault defined for all protocols.  The default value is 87380 bytes.  (On Linux 2.4, this will be lowered to 43689 in low-memory systems.)   If  larger  re‐
                     ceive  buffer sizes are desired, this value should be increased (to affect all sockets).  To employ large TCP windows, the net.ipv4.tcp_window_scaling must
                     be enabled (default).

              max    the maximum size of the receive buffer used by each TCP socket.  This value does not override the global net.core.rmem_max.  This is not used to limit  the
                     size of the receive buffer declared using SO_RCVBUF on a socket.  The default value is calculated using the formula

                         max(87380, min(4 MB, tcp_mem[1]*PAGE_SIZE/128))

                     (On Linux 2.4, the default is 87380*2 bytes, lowered to 87380 in low-memory systems).

```

### Examples

#### Example 1: Without SO_RCVBUF configured

In this case, the default socket receive buffer will be set up by using the default value of `/proc/sys/net/ipv4/tcp_rmem` then Linux will adjust the receive buffer accordingly based on the usage, the max receive buffer allocated would be the max value of `/proc/sys/net/ipv4/tcp_rmem`.

```
/proc/sys/net/ipv4/tcp_rmem: 4096	131072	6291456

$ nc -vvv www.pinterest.com 443
[in another terminal]
$ ss -mtp
...
ESTAB              0                   0                                            192.168.1.102:45592                                      184.51.48.250:https                      users:(("nc",pid=627316,fd=3))
	 skmem:(r0,rb131072,t0,tb87040,f0,w0,o0,bl0,d1)                                                                                                     

...
```

The receive buffer value is 131072 bytes, which is the default OS receive buffer.

#### Example 2: With SO_RCVBUF configured

In this case, the actual receive buffer would be the 2 x configured value. However, it cannot be greater than the value of `/proc/sys/net/core/rmem_max`.

##### Example 2.1: Set up receive buffer value 100000 bytes

```
/proc/sys/net/core/rmem_max: 212992

$ netcat -v -I 100000 www.pinterest.com 443
[in another terminal]
$ ss -mtp
...
ESTAB              0                   0                                            192.168.1.102:33106                                       146.75.92.84:https                      users:(("netcat",pid=687860,fd=3))
	 skmem:(r0,rb200000,t0,tb46080,f0,w0,o0,bl0,d1)
...
```

The actual receive buffer is 100000 x 2 == 200000 bytes.

##### Example 2.2: Set up receive buffer value greater than the max value

```
/proc/sys/net/core/rmem_max: 212992

$ netcat -v -I 500000 www.pinterest.com 443
[in another terminal]
$ ss -mtp
...
ESTAB              0                   0                                            192.168.1.102:42418                                     151.101.188.84:https                      users:(("netcat",pid=687953,fd=3))
	 skmem:(r0,rb425984,t0,tb46080,f0,w0,o0,bl0,d1)
...
```

The actual receive buffer is the max value 212992 x 2 == 425984 bytes. Even the configured value is 500000 bytes but it's been capped by the max value of the global setting `/proc/sys/net/core/rmem_max`.
