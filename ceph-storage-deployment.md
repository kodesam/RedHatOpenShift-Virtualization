
```markdown
# Red Hat Ceph Storage Deployment

Please ensure that before deployment, all hosts can communicate with each other via SSH seamlessly, without requiring manual password entry or key approval.

## Enable Required Subscriptions

```bash
subscription-manager repos --enable=rhel-8-for-x86_64-baseos-rpms --enable=ansible-2.9-for-rhel-8-x86_64-rpms --enable=rhceph-5-tools-for-rhel-8-x86_64-rpms --enable rhel-8-for-x86_64-appstream-rpms
```

## Install Required Packages

```bash
dnf install podman ansible cephadm-ansible cephadm -y
```

## Login to the Registry

```bash
podman login -u user -p password registry.redhat.io
```

## Create Hosts File

Create a file containing a list of the hosts for the Red Hat Ceph Storage cluster:

```plaintext
Host1.domain.name
Host2.domain.name
Host3.domain.name
```

## Pre-Deployment Test

```bash
cd /usr/share/cephadm-ansible/
ansible-playbook -i /path/to/hosts_file cephadm-preflight.yml --extra-vars "ceph_origin=rhcs"
```

## Copy Configuration Files

The files below will need to be copied to any other host that is a member of the Red Hat Ceph Storage cluster. Make sure that the files will be copied to the same path on the other hosts (`/etc/ceph/`):

- `/etc/ceph/ceph.client.admin.keyring`
- `/etc/ceph/ceph.conf`

## Create Registry Login File

Create a file containing the registry file login details:

```json
{
  "url": "registry.redhat.io",
  "username": "email@user.name",
  "password": "my-password"
}
```

## Start Deployment

```bash
cephadm bootstrap --mon-ip bond_ip_from_any_cluster_member --registry-json /etc/ceph/ceph_login_details_file --allow-fqdn-hostname --yes-i-know --cluster-network bond_subnet_mask # e.g 192.168.0.0/16
```

## Add Host Roles

Once the deployment is successfully completed, we can add all the host's roles in the cluster. The simplest way to do it is through cephadm shell. Here is an example:

```bash
cephadm shell
ceph orch host add Host1.domain.name bond_ip --labels=osd,mgr,dashboard
ceph orch host add Host2.domain.name bond_ip --labels=osd,mgr
ceph orch host add Host3.domain.name bond_ip --labels=osd,mon
ceph orch host add Host4.domain.name bond_ip --labels=osd,mon
ceph orch host add Host5.domain.name bond_ip --labels=osd,grafana
ceph orch host add Host6.domain.name bond_ip --labels=osd,prometheus
ceph orch host add Host7.domain.name bond_ip --labels=osd,mdss
ceph orch host add Host8.domain.name bond_ip --labels=osd,mdss
ceph orch host add Host9.domain.name bond_ip --labels=osd
```

Make sure to follow best practices for high availability (HA) and performance, which can vary depending on cluster hardware and the number of hosts in the cluster.

# Red Hat Ceph Storage Setup

## Pool Creation

This section describes the Red Hat Ceph Storage-specific tuning performed on the Red Hat Ceph Storage nodes to cater to this large-scale environment. On this specific setup, we created a single pool for NVME and SSD disks:

```bash
ceph osd pool create ocp_pool
```

Note that for spinning disks, it is highly recommended to create a separate pool in order to optimize the cluster performance. We can do that by creating crush roles:

```bash
ceph osd crush rule create-replicated replicated_hdd default host hdd
ceph osd crush rule create-replicated replicated_ssd default host ssd
```

And then applying those rules to the pools, for example:

```bash
ceph osd pool set hdd_pool crush_rule replicated_hdd
```

Or:

```bash
ceph osd pool set nvme_pool crush_rule replicated_ssd
```

## Placement Groups Tuning

We can achieve the optimal number of PGs per pool by setting our target at 100 PGs per OSD (according to best practice for rbd & librados), then multiply by the maximum used capacity of the pool (default is 85%), divide by the number of replicas, and round to the nearest power of 2 - 2^(round(log2(x))). For this cluster, that was 8192:

```bash
ceph osd pool set ocp_pool pg_autoscaler_mode off
ceph osd pool set ocp_pool pg_num 8192
```

For example, for a setup with 200 disks or OSDs:
(100 * 200 * 0.85) / 3 = rounded to a power of 2 is 4096 total PGs.

We scripted it using `bc`:

```bash
echo "x=l(100*200*0.85/3)/l(2); scale=0; 2^((x+0.5)/1)" | bc -l
```

Note that we can increase the number of PGs per OSD even further. That can potentially reduce the variance in per-OSD load across the cluster, but each PG requires a bit more CPU and memory on the OSDs that are storing it. Therefore, the number of OSDs should be tested and tuned per environment.

## Prometheus Tuning

For monitoring the Red Hat Ceph Storage cluster, we can use the Red Hat Ceph Storage dashboard. To enable stats to display for the pool, we can run:

```bash
ceph config set mgr mgr/prometheus/rbd_stats_pools ocp_pool
```

To lessen the load on the system for large clusters, we can throttle the pool stats collection with:

```bash
ceph config set mgr mgr/prometheus/rbd_stats_pools_refresh_interval 600
```

It's also a good idea to lower the polling rate for Prometheus in order to avoid turning the ceph manager into a bottleneck, which may result in other ceph-mgr plugins not getting time to run.

In this case, the following command sets the scraping interval to 60 seconds:

```bash
ceph config set mgr mgr/prometheus/scrape_interval 60
```
