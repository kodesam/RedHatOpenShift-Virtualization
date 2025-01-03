
Here's the content formatted in Markdown for the `tuned-profile-config.md` file:

```markdown
# Steps for TuneD Profile Configuration

## TuneD Profile

TuneD was used to optimize the network performance for this scale. TuneD can be installed via:

```bash
dnf install tuned -y; systemctl start tuned; systemctl enable tuned
```

In our environment, we changed the parameters as shown in the screenshot below.

```ini
[main]
summary=Optimize for RHCS performance focused on low latency network performance

[vm]
# Disable Transparent Huge Pages (Default: always)
transparent_hugepages=never

[sysctl]
# Network core: Adjust the busy read threshold to 50 (Default: 0)
net.core.busy_read=50
# Network core: Adjust the busy poll threshold to 50 (Default: 0)
net.core.busy_poll=50
# NUMA balancing: Disable NUMA balancing (Default: 1)
kernel.numa_balancing=0
# Kernel task timeout: Set hung task timeout to 600 seconds (Default: 120)
kernel.hung_task_timeout_secs=600
# NMI watchdog: Disable the NMI watchdog (Default: 1)
kernel.nmi_watchdog=0
# Virtual Memory statistics interval: Set interval to 10 seconds (Default: 1)
vm.stat_interval=10
# Kernel timer migration: Disable kernel timer migration (Default: 1)
kernel.timer_migration=0
# ARP cache tuning: Threshold 1 for garbage collector triggering (Default: 128)
net.ipv4.neigh.default.gc_thresh1 = 4096
# ARP cache tuning: Threshold 2 for garbage collector triggering (Default: 512)
net.ipv4.neigh.default.gc_thresh2 = 16384
# ARP cache tuning: Threshold 3 for garbage collector triggering (Default: 1024)
net.ipv4.neigh.default.gc_thresh3 = 32768
# ARP Flux: Enable ARP filtering (Default: 0)
net.ipv4.conf.all.arp_filter = 1
# ARP Flux: Ignore ARP requests from unknown sources (Default: 0)
net.ipv4.conf.all.arp_ignore = 1
# ARP Flux: Announce local source IP address on ARP requests (Default: 0)
net.ipv4.conf.all.arp_announce = 1
# TCP/IP Tuning: Enable TCP window scaling (Default: 0)
net.ipv4.tcp_window_scaling = 1
# TCP Fast Open: Enable TCP Fast Open (Default: 1)
net.ipv4.tcp_fastopen = 3
# Buffer Size Tuning: Maximum receive buffer size for all network interfaces (Default: 212992)
net.core.rmem_max = 11639193
# Buffer Size Tuning: Maximum send buffer size for all network interfaces (Default: 212992)
net.core.wmem_max = 11639193
# Buffer Size Tuning: Default send buffer size for all network interfaces (Default: 212992)
net.core.wmem_default = 2909798
# NIC buffers: Maximum number of packets per network device queue (Default: 300)
net.core.netdev_budget = 1000
# NIC buffers: Maximum backlog size for incoming packets (Default: 1000)
net.core.netdev_max_backlog = 5000

[bootloader]
cmdline_network_latency=skew_tick=1 tsc=reliable rcupdate.rcu_normal_after_boot=1
```

## Key Modifications

### Hugepages
```ini
transparent_hugepages=never
```
Disabling THP ensures consistent RHCS performance, avoiding potential slowdowns caused by automatic page size handling.

### Network Busy Read Threshold
```ini
net.core.busy_read=50
```
Optimizes network performance during high-demand periods.

### Network Busy Poll Threshold
```ini
net.core.busy_poll=50
```
Enhances network responsiveness during peak traffic.

### NUMA Balancing
```ini
kernel.numa_balancing=0
```
Disabling NUMA balancing minimizes latency and enhances performance.

### Hung Task Timeout
```ini
kernel.hung_task_timeout_secs=600
```
Allows more time for task recovery, improving system stability.

### NMI Watchdog
```ini
kernel.nmi_watchdog=0
```
Reduces disruptive interruptions, prioritizing tasks.

### Virtual Memory Statistics Interval
```ini
vm.stat_interval=10
```
Facilitates more frequent monitoring of memory-related metrics.

### Timer Migration
```ini
kernel.timer_migration=0
```
Ensures timers remain on their original CPU cores, minimizing delays.

### ARP Cache
```ini
net.ipv4.neigh.default.gc_thresh1 = 8192  
net.ipv4.neigh.default.gc_thresh2 = 32768 
net.ipv4.neigh.default.gc_thresh3 = 65536
```
Increases ARP cache size to accommodate more entries, improving efficiency.

### ARP Flux
```ini
net.ipv4.conf.all.arp_filter=1 
net.ipv4.conf.all.arp_ignore=1 
net.ipv4.conf.all.arp_announce=1
```
Optimizes network operations and ensures efficient communication.

### TCP Window Scaling
```ini
net.ipv4.tcp_window_scaling=1
```
Enables larger TCP window size for better use of high-bandwidth networks.

### TCP Handshake Tuning
```ini
net.ipv4.tcp_fastopen = 3
```
Enables TCP Fast Open for both client and server applications, reducing connection establishment time.

Once the TuneD profile is initially configured, next is to tackle the buffer size settings for the RHCS network.
```

Feel free to adjust the content as needed! If you have any questions or need further assistance, just let me know.
