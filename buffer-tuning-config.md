
```markdown
# Buffer Size Tuning for Red Hat Ceph Storage Network

## Introduction

The next step will be to calculate the socket “send buffer size” and “receive buffer size.” Generally speaking, each socket's read/write buffer can hold either a minimum of 2 packets, a default of 4 packets, or a maximum of 10 packets. If the network socket buffer is too small it might fill up and reduce the effective throughput which will impact performance. If the network socket buffer is set large enough it can improve performance to a certain extent.

This resource will explore the optimal buffer size for achieving maximum throughput in a single TCP connection.

## What You Will Learn

- Optimal buffer tuning settings for this project

## Prerequisites

- Complete initial tuned profile configuration

## Buffer Size Tuning

```ini
net.core.rmem_max=11639193
net.core.wmem_max=11639193
net.core.wmem_default=2909798
```

The following formula calculates optimal size:

```plaintext
(round trip delay in microseconds) x (size of the link in Mb/s) x 1024^2
```

For instance, if the bond is operating at 100000 Mb/s and the latency from the RHCS node to the OpenShift cluster is 0.208 (converted to microseconds by dividing by 1000), the calculation should be as follows:

```plaintext
Optimal size = 0.208 (microseconds) x 100000 (Mb/s) x 1024^2 = 10.4 (converted to bytes) = 11639193
```

Alternatively, we can use the ping command with specific parameters to measure latency and calculate the optimal buffer size:

```bash
ping -I bond0 -c 60 -q 192.168.216.90 | grep avg | awk -F "/" '{printf "%f", ($5 / 1000) * 100000 * 1024^2}'
```

The above command sends 60 ping packets through the bond0 interface to the specified Node IP address (192.168.216.90), and then extracts the average latency value. The latency is multiplied by 100000 and 1024^2 to calculate the optimal buffer size in bytes.

Now, all that remains is to configure the `wmem_default` which contains 4 packets, meaning ¼ of 11639193 which is 2909798.

## NIC Buffers

```ini
net.core.netdev_budget=1000
```

On large-scale setups that contain multiple hosts, the rate of incoming traffic could potentially exceed the kernel capability to drain the buffers fast enough and if that happens the NIC buffers will overflow and the traffic will be lost.

If that happens, the lost traffic will be counted as softirq misses. We can increase the CPU time for the softirq to avoid that scenario, this is known as the `netdev_budget`, and we can increase the budget as necessary. On our setup, we increased the budget to 1000, which means that the softirq will drain 1000 messages on the NIC before getting off the CPU.

If the 3rd column in `/proc/net/softnet_stat` is gradually increasing over time:

```plaintext
01877e29 00000000 00000022 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 0000005d
0c4a6107 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 0000005e
01d05820 00000000 00000012 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 0000005f
092b933a 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000060
```

This indicates that the softirq did not get enough CPU time, in that case, the budget can be increased, preferably with small increments.

## Backlog Queue

```ini
net.core.netdev_max_backlog=5000
```

Within the Linux kernel, there is a queue where traffic is stored after it was received from the NIC, but before it got processed by one of the protocol stacks (TCP/IP/ISCSI).

Each CPU core has a backlog queue where traffic is stored, if the queue is already at its maximum capacity any additional packets will get dropped.

If the 2nd column in `/proc/net/softnet_stat` is gradually increasing over time:

```plaintext
04f88d2c 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000
023a354d 00000000 00000018 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000001
10df99e1 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000002
01ba2dec 00000000 00000011 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000003
```

This indicates that the netdev backlog queue overflows and `netdev_max_backlog` need to be increased, again preferably with small increments.

In this scale setup, we set the value to 5000.

## Network Latency and Timing

```ini
[bootloader]
cmdline_network_latency=skew_tick=1 tsc=reliable rcupdate.rcu_normal_after_boot=1
```

The above bootloader setting enables the kernel to dynamically adjust its timer ticks, which are small fixed time intervals, to account for network latency variations. This adjustment helps the operating system to better cope with network delays, improving overall system performance and responsiveness when dealing with varying network latencies.

## Port Bond Settings

The following four hosts ports are designated to serve as the dedicated Red Hat® Ceph® Storage network bond within the RHCS cluster, we created them using the following steps:

### Creating the Network Bond

```bash
nmcli connection add type bond ifname bond0 bond.options "mode=balance-alb"
```

### Adding All the 25 Gbps Interfaces to the Bond

```bash
nmcli connection add type ethernet slave-type bond con-name bond0-port1 ifname ens3f0 master bond0
nmcli connection add type ethernet slave-type bond con-name bond0-port2 ifname ens3f1 master bond0
nmcli connection add type ethernet slave-type bond con-name bond0-port3 ifname ens2f0 master bond0
nmcli connection add type ethernet slave-type bond con-name bond0-port4 ifname ens2f1 master bond0
```

Now that we have a single 100 Gbps bond interface, we need to make sure the main bond IP won’t change due to DHCP; a simple solution is to set one interface as primary and then set it to active, in this case, we choose `ens2f0`:

```bash
nmcli dev mod bond0 +bond.options "primary=ens3f0"
```

Now we can move on to the RHCS deployment.
```

Feel free to adjust the content as needed! If you have any questions or need further assistance, just let me know.
