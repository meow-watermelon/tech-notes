# Soft-RoCE Configuration

## Intro

This document illustrates how to set up Soft-RoCE device and how to test the configured RDMA device.

## Set up Soft-RoCE Device

Soft-RoCE is the software implementation of the RDMA transport. Soft-RoCE does not require special Ethernet devices. In the following example, we would use the loopback device(`lo`) to configure a Soft-RoCE device on it.

1. Install the `rdma-core` package. If we need to do further tests and developments, please install following packages as well:

```
rdma-core-devel
infiniband-diags
libibverbs
libibverbs-utils
```

2. Load the `rdma_rxe` kernel module:

```
# modprobe rdma_rxe
# lsmod | grep rdma_rxe
rdma_rxe              217088  0
ib_uverbs             208896  2 rdma_rxe,rdma_ucm
ip6_udp_tunnel         16384  1 rdma_rxe
udp_tunnel             36864  1 rdma_rxe
ib_core               573440  12 rdma_cm,ib_ipoib,rdma_rxe,rpcrdma,ib_srpt,iw_cm,ib_iser,ib_umad,ib_isert,rdma_ucm,ib_uverbs,ib_cm
```

3. Create a Soft-RoCE device via the `rdma` command. This command is a part of `iproute` package.

```
# rdma link add rxe0 type rxe netdev lo
```

Once the `rxe0` device is created, we can use the following command to show the device:

```
# rdma link
link rxe0/1 state ACTIVE physical_state LINK_UP netdev lo
```

4. To delete the `rxe0` device, please use the foolowing command:

```
# rdma link del rxe0
```

## Test Soft-RoCE Device

In this section, we use `ibv_rc_pingpong` to test the `rxe0` device. Before triggering the actual test, we need to figure out the deivce GID we need to use. The utility `ibv_rc_pingpong` will be listening on an IPv4 address with port `18515` by default, and the GID is a 128-bit address modeled after IPv6 addresses. So we need to pick up the correct GID index to be used.

1. Check the correct GID index we shall use for the command `ibv_rc_pingpong`:

```
# ibv_devinfo -d rxe0 -i 1 -v | grep GID
			GID[  0]:		fe80::200:ff:fe00:0, RoCE v2
			GID[  1]:		::ffff:127.0.0.1, RoCE v2
			GID[  2]:		::1, RoCE v2
```

In the above example, `-d` specifies the device name, `-i` specifies the port number, `-v` shows verbose output. Based on the output we get, we shall use GID 1 as it has the correct IPv4 address represented.

2. Initiate a server by following command:

```
# ibv_rc_pingpong -g 1 -d rxe0 -i 1
  local address:  LID 0x0000, QPN 0x000012, PSN 0x55c1a0, GID ::ffff:127.0.0.1
```

3. Start a client in another terminal to trigger the test by following command:

```
# ibv_rc_pingpong -g 1 -d rxe0 -i 1 127.0.0.1
  local address:  LID 0x0000, QPN 0x000013, PSN 0x2227ab, GID ::ffff:127.0.0.1
  remote address: LID 0x0000, QPN 0x000012, PSN 0x55c1a0, GID ::ffff:127.0.0.1
8192000 bytes in 0.04 seconds = 1522.04 Mbit/sec
1000 iters in 0.04 seconds = 43.06 usec/iter
```

And on the server side:

```
# ibv_rc_pingpong -g 1 -d rxe0 -i 1
  local address:  LID 0x0000, QPN 0x000012, PSN 0x55c1a0, GID ::ffff:127.0.0.1
  remote address: LID 0x0000, QPN 0x000013, PSN 0x2227ab, GID ::ffff:127.0.0.1
8192000 bytes in 0.04 seconds = 1521.90 Mbit/sec
1000 iters in 0.04 seconds = 43.06 usec/iter
```

## References

[Verify that RDMA is working](https://www.rdmamojo.com/2015/01/24/verify-rdma-working/)

[Soft-RoCE Requires a Specific IPv6 Address](https://transactional.blog/rdma/soft-roce-requires-ipv6-link-local)

[rdma_rxe usage problem](https://www.spinics.net/lists/linux-rdma/msg108341.html)
