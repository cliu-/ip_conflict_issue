## Story

This issue was found by @Eng_A on June 19, 2017. When he was trying to update applications on promo environment, `authservice` which is an authentication micro service in our SaaS system reported that it could not connect to postgres database, and new service instance could not be created.

I tried to use `pgAdmin` to connect postgres database but failed because of connection establishing error, I tried to ping the delegate IP address of database (since we're using pgpool as proxy), the address was not reachable, while the real IP addresses of pgpool are pingable.

Since the issue blocked applications' promotion, I simply logged into the mast pgpool node over SSH and restarted pgpool job, delegate IP address floated to another pgpool node and postgres service was available right away. However, this issue was observed every time when we were going to do something on promo environment.

## Postgres Service Architecture

We're using postgres as backend database for most of our micro services, especially for those platform level micro services. And these micro services share one postgres cluster but have their own database.

Our postgres cluster supports multiple servers, there is one primary and be replicated by standby over WAL streaming.

We adopt pgpool as connection pool, failover performer, and load balancer (not at SQL level) at front of postgres cluster. pgpool supports high availability with its sub-project `watchdog` which can generate a virtual IP address to master node. Master pgpool node could be switched over within pgpool cluster, and the virtual IP address is always attached on master node. The virtual IP address is the connection point to all database clients.

On promo environment, we have two postgres server node, and two pgpool nodes.

## Diagnosing

#### Day 1

I firstly checked log file on `pgpool_0` host. From log messages, we can see the issue happened at `22:45:10`, `pgpool_0` was master node. At this moment, the socket connection between `pgpool_0` and `pgpool_1` had been `reset by peer`, and then `pgpool_0` lost its standby `pgpool_1`. Because `pgpool_0` was master, this standby lost would affect nothing, the virtual IP address was still being attached to `pgpool_0`.

```
/var/log/pgpool.log @ pgpool_0(10.8.52.190):
Jun 20 22:42:24 localhost pgpool[17603]: [209-1] 2017-06-20 22:42:24: pid 17603: LOG:  new IPC connection received
Jun 20 22:42:24 localhost pgpool[17603]: [210-1] 2017-06-20 22:42:24: pid 17603: LOG:  received node status change ipc message
Jun 20 22:42:24 localhost pgpool[17603]: [210-2] 2017-06-20 22:42:24: pid 17603: DETAIL:  Heartbeat signal found
Jun 20 22:45:10 localhost pgpool[17603]: [211-1] 2017-06-20 22:45:10: pid 17603: LOG:  read from socket failed with error :"Connection reset by peer"
Jun 20 22:45:10 localhost pgpool[17603]: [212-1] 2017-06-20 22:45:10: pid 17603: LOG:  client socket of Linux_fd5f70a6-3e7e-452e-8fd2-7b0e91bb6193_9999 is closed
Jun 20 22:45:15 localhost pgpool[17603]: [213-1] 2017-06-20 22:45:15: pid 17603: LOG:  read from socket failed, remote end closed the connection
Jun 20 22:45:15 localhost pgpool[17603]: [214-1] 2017-06-20 22:45:15: pid 17603: LOG:  outbound socket of Linux_fd5f70a6-3e7e-452e-8fd2-7b0e91bb6193_9999 is closed
Jun 20 22:45:15 localhost pgpool[17603]: [215-1] 2017-06-20 22:45:15: pid 17603: LOG:  remote node "Linux_fd5f70a6-3e7e-452e-8fd2-7b0e91bb6193_9999" is not reachable
Jun 20 22:45:15 localhost pgpool[17603]: [215-2] 2017-06-20 22:45:15: pid 17603: DETAIL:  marking the node as lost
Jun 20 22:45:15 localhost pgpool[17603]: [216-1] 2017-06-20 22:45:15: pid 17603: LOG:  remote node "Linux_fd5f70a6-3e7e-452e-8fd2-7b0e91bb6193_9999" is lost
Jun 20 22:45:30 localhost pgpool[17603]: [217-1] 2017-06-20 22:45:30: pid 17603: LOG:  new watchdog node connection is received from "10.8.53.191:36513"
Jun 20 22:45:30 localhost pgpool[17603]: [218-1] 2017-06-20 22:45:30: pid 17603: NOTICE:  New node joined the cluster hostname:"10.8.53.191" port:9000 pgpool_port:9999
Jun 20 22:45:33 localhost pgpool[17603]: [219-1] 2017-06-20 22:45:33: pid 17603: LOG:  new outbond connection to 10.8.53.191:9000 
```

Next, I checked log file on `pgpool_1` host to see what's going. At `22:45:15`, this standby node was announcing the master was dead, and then escalated itself to master and brought up virtual IP address. Later, at `22:45:20`, when the main pgpool process `17510` was trying to check backend databases' health, it had a failure: `connection to host:"10.8.53.131:5432" failed`. Here something interesting was happening at `22:45:30`, this new master, `pgpool_1`, received a heartbeat from its predecessor, that's really embarrassing. So, anyway, the watchdog processor `17513` needed to de-escalate itself to standby and brought down the virtual IP address.

After all of these, the virtual IP address became unreachable on `pgpool_0`, because mac-port-mapping cache had been updated on switch by `pgpool_1` when it's bringing up virtual IP address.

~~~
/var/log/pgpool.log @ pgpool_1(10.8.52.191):
Jun 20 22:42:21 localhost pgpool[17515]: [12-1] 2017-06-20 22:42:21: pid 17515: LOG:  creating watchdog heartbeat receive socket.
Jun 20 22:42:21 localhost pgpool[17515]: [12-2] 2017-06-20 22:42:21: pid 17515: DETAIL:  set SO_REUSEPORT
Jun 20 22:43:10 localhost pgpool[17514]: [11-1] 2017-06-20 22:43:10: pid 17514: LOG:  watchdog: lifecheck started
Jun 20 22:45:15 localhost pgpool[17514]: [12-1] 2017-06-20 22:45:15: pid 17514: LOG:  informing the node status change to watchdog
Jun 20 22:45:15 localhost pgpool[17514]: [12-2] 2017-06-20 22:45:15: pid 17514: DETAIL:  node id :1 status = "NODE DEAD" message:"No heartbeat signal from node"
Jun 20 22:45:15 localhost pgpool[17513]: [22-1] 2017-06-20 22:45:15: pid 17513: LOG:  new IPC connection received
Jun 20 22:45:15 localhost pgpool[17513]: [23-1] 2017-06-20 22:45:15: pid 17513: LOG:  received node status change ipc message
Jun 20 22:45:15 localhost pgpool[17513]: [23-2] 2017-06-20 22:45:15: pid 17513: DETAIL:  No heartbeat signal from node
Jun 20 22:45:15 localhost pgpool[17513]: [24-1] 2017-06-20 22:45:15: pid 17513: LOG:  remote node "Linux_caf01faa-9ce7-408c-a326-ab1e552afcb9_9999" is lost
Jun 20 22:45:15 localhost pgpool[17513]: [25-1] 2017-06-20 22:45:15: pid 17513: LOG:  watchdog cluster has lost the coordinator node
Jun 20 22:45:15 localhost pgpool[17513]: [26-1] 2017-06-20 22:45:15: pid 17513: LOG:  watchdog node state changed from [STANDBY] to [JOINING]
Jun 20 22:45:19 localhost pgpool[17513]: [27-1] 2017-06-20 22:45:19: pid 17513: LOG:  watchdog node state changed from [JOINING] to [INITIALIZING]
Jun 20 22:45:20 localhost pgpool[17513]: [28-1] 2017-06-20 22:45:20: pid 17513: LOG:  I am the only alive node in the watchdog cluster
Jun 20 22:45:20 localhost pgpool[17513]: [28-2] 2017-06-20 22:45:20: pid 17513: HINT:  skiping stand for coordinator state
Jun 20 22:45:20 localhost pgpool[17513]: [29-1] 2017-06-20 22:45:20: pid 17513: LOG:  watchdog node state changed from [INITIALIZING] to [MASTER]
Jun 20 22:45:20 localhost pgpool[17513]: [30-1] 2017-06-20 22:45:20: pid 17513: LOG:  I am announcing my self as master/coordinator watchdog node
Jun 20 22:45:20 localhost pgpool[17618]: [12-1] 2017-06-20 22:45:20: pid 17618: LOG:  trying connecting to PostgreSQL server on "10.8.53.131:5432" by INET socket
Jun 20 22:45:20 localhost pgpool[17618]: [12-2] 2017-06-20 22:45:20: pid 17618: DETAIL:  timed out. retrying...
Jun 20 22:45:20 localhost pgpool[17510]: [13-1] 2017-06-20 22:45:20: pid 17510: LOG:  failed to connect to PostgreSQL server on "10.8.53.131:5432", timed out
Jun 20 22:45:20 localhost pgpool[17510]: [14-1] 2017-06-20 22:45:20: pid 17510: ERROR:  failed to make persistent db connection
Jun 20 22:45:20 localhost pgpool[17510]: [14-2] 2017-06-20 22:45:20: pid 17510: DETAIL:  connection to host:"10.8.53.131:5432" failed
Jun 20 22:45:24 localhost pgpool[17513]: [31-1] 2017-06-20 22:45:24: pid 17513: LOG:  I am the cluster leader node
Jun 20 22:45:24 localhost pgpool[17513]: [31-2] 2017-06-20 22:45:24: pid 17513: DETAIL:  our declare coordinator message is accepted by all nodes
Jun 20 22:45:24 localhost pgpool[17513]: [32-1] 2017-06-20 22:45:24: pid 17513: LOG:  I am the cluster leader node. Starting escalation process
Jun 20 22:45:24 localhost pgpool[17513]: [33-1] 2017-06-20 22:45:24: pid 17513: LOG:  escalation process started with PID:17760
Jun 20 22:45:24 localhost pgpool[17760]: [33-1] 2017-06-20 22:45:24: pid 17760: LOG:  watchdog: escalation started
Jun 20 22:45:25 localhost pgpool[17514]: [13-1] 2017-06-20 22:45:25: pid 17514: LOG:  informing the node status change to watchdog
Jun 20 22:45:25 localhost pgpool[17514]: [13-2] 2017-06-20 22:45:25: pid 17514: DETAIL:  node id :1 status = "NODE ALIVE" message:"Heartbeat signal found"
Jun 20 22:45:25 localhost pgpool[17513]: [34-1] 2017-06-20 22:45:25: pid 17513: LOG:  new IPC connection received
Jun 20 22:45:25 localhost pgpool[17513]: [35-1] 2017-06-20 22:45:25: pid 17513: LOG:  received node status change ipc message
Jun 20 22:45:25 localhost pgpool[17513]: [35-2] 2017-06-20 22:45:25: pid 17513: DETAIL:  Heartbeat signal found
Jun 20 22:45:28 localhost pgpool[17760]: [34-1] 2017-06-20 22:45:28: pid 17760: LOG:  watchdog bringing up delegate IP, 'if_up_cmd' succeeded
Jun 20 22:45:28 localhost pgpool[17513]: [36-1] 2017-06-20 22:45:28: pid 17513: LOG:  watchdog escalation process with pid: 17760 exit with SUCCESS.
Jun 20 22:45:30 localhost pgpool[17513]: [37-1] 2017-06-20 22:45:30: pid 17513: LOG:  new outbond connection to 10.8.53.190:9000 
Jun 20 22:45:30 localhost pgpool[17513]: [38-1] 2017-06-20 22:45:30: pid 17513: WARNING:  "Linux_fd5f70a6-3e7e-452e-8fd2-7b0e91bb6193_9999" is the coordinator as per our record but "Linux_caf01faa-9ce7-408c-a326-ab1e552afcb9_9999" is also announcing as a coordinator
Jun 20 22:45:30 localhost pgpool[17513]: [38-2] 2017-06-20 22:45:30: pid 17513: DETAIL:  re-initializing the cluster
Jun 20 22:45:30 localhost pgpool[17513]: [39-1] 2017-06-20 22:45:30: pid 17513: LOG:  watchdog node state changed from [MASTER] to [JOINING]
Jun 20 22:45:30 localhost pgpool[17783]: [39-1] 2017-06-20 22:45:30: pid 17783: LOG:  watchdog: de-escalation started
Jun 20 22:45:30 localhost pgpool[17513]: [40-1] 2017-06-20 22:45:30: pid 17513: LOG:  watchdog node state changed from [JOINING] to [INITIALIZING]
Jun 20 22:45:30 localhost pgpool[17618]: [13-1] 2017-06-20 22:45:30: pid 17618: LOG:  trying connecting to PostgreSQL server on "10.8.53.131:5432" by INET socket
Jun 20 22:45:30 localhost pgpool[17618]: [13-2] 2017-06-20 22:45:30: pid 17618: DETAIL:  timed out. retrying...
Jun 20 22:45:30 localhost pgpool[17510]: [15-1] 2017-06-20 22:45:30: pid 17510: LOG:  failed to connect to PostgreSQL server on "10.8.53.131:5432", timed out
Jun 20 22:45:30 localhost pgpool[17510]: [16-1] 2017-06-20 22:45:30: pid 17510: ERROR:  failed to make persistent db connection
Jun 20 22:45:30 localhost pgpool[17510]: [16-2] 2017-06-20 22:45:30: pid 17510: DETAIL:  connection to host:"10.8.53.131:5432" failed
Jun 20 22:45:30 localhost pgpool[17510]: [17-1] 2017-06-20 22:45:30: pid 17510: LOG:  setting backend node 0 status to NODE DOWN
Jun 20 22:45:30 localhost pgpool[17510]: [18-1] 2017-06-20 22:45:30: pid 17510: LOG:  received degenerate backend request for node_id: 0 from pid [17510]
Jun 20 22:45:30 localhost pgpool[17513]: [41-1] 2017-06-20 22:45:30: pid 17513: LOG:  new IPC connection received
Jun 20 22:45:30 localhost pgpool[17513]: [42-1] 2017-06-20 22:45:30: pid 17513: NOTICE:  error processing IPC from socket
Jun 20 22:45:30 localhost pgpool[17513]: [43-1] 2017-06-20 22:45:30: pid 17513: NOTICE:  error writing to IPC socket
Jun 20 22:45:30 localhost pgpool[17510]: [19-1] 2017-06-20 22:45:30: pid 17510: LOG:  degenerate backend request for 1 node(s) from pid [17510] is canceled by other pgpool
Jun 20 22:45:31 localhost pgpool[17513]: [44-1] 2017-06-20 22:45:31: pid 17513: LOG:  watchdog node state changed from [INITIALIZING] to [STANDBY]
Jun 20 22:45:31 localhost pgpool[17513]: [45-1] 2017-06-20 22:45:31: pid 17513: LOG:  successfully joined the watchdog cluster as standby node
Jun 20 22:45:31 localhost pgpool[17513]: [45-2] 2017-06-20 22:45:31: pid 17513: DETAIL:  our join coordinator request is accepted by cluster leader node "Linux_caf01faa-9ce7-408c-a326-ab1e552afcb9_9999"
Jun 20 22:45:33 localhost pgpool[17513]: [46-1] 2017-06-20 22:45:33: pid 17513: LOG:  new watchdog node connection is received from "10.8.53.190:62668"
Jun 20 22:45:33 localhost pgpool[17513]: [47-1] 2017-06-20 22:45:33: pid 17513: NOTICE:  New node joined the cluster hostname:"10.8.53.190" port:9000 pgpool_port:9999
Jun 20 22:45:36 localhost pgpool[17783]: [40-1] 2017-06-20 22:45:36: pid 17783: WARNING:  watchdog bringing down delegate IP, if_down_cmd failed
Jun 20 22:45:36 localhost pgpool[17783]: [41-1] 2017-06-20 22:45:36: pid 17783: WARNING:  watchdog de-escalation failed to bring down delegate IP
Jun 20 22:45:36 localhost pgpool[17513]: [48-1] 2017-06-20 22:45:36: pid 17513: LOG:  watchdog de-escalation process with pid: 17783 exit with SUCCESS.
Jun 20 22:45:40 localhost pgpool[17618]: [14-1] 2017-06-20 22:45:40: pid 17618: LOG:  trying connecting to PostgreSQL server on "10.8.53.131:5432" by INET socket
Jun 20 22:45:40 localhost pgpool[17618]: [14-2] 2017-06-20 22:45:40: pid 17618: DETAIL:  timed out. retrying...
~~~

 This is a typical split-brain syndrome in pgpool cluster, there is a network partition happened on `pgpool_1`, but recovered very quickly. The question is why `pgpool_1` would be isolated in network?

The initial idea was that the network in our lab was not stable, that would cause a quick network partition on `pgpool_1`. To verify the idea, I did below things:

- Checked if the issue happened on other environments (CI and QE) which are in the same network.
- Run `icpld` on both two pgpool hosts to monitor (at 5-second interval) the network status.

The result was that we did not find the same issue on CI or QE, and after overnight monitoring, `icpld` did not report any network outage or break.

###### Day 1 summary:

- The issue happens frequently, this is good for diagnosing.
- The issue only happened on promo environment.
- one phenomenon of this issue is split-brain syndrome in pgpool cluster.
- only `pgpool_1` (`10.8.52.191`) is affected but `pgpool_0` (`10.8.52.190`) is reachable.
- there is no network break or outage happened in lab.

#### Day 2

I went back to log files which had accumulated couple hours messages from pgpool. I examined them carefully and finally found an interesting thing: the issue repeats periodically, and the periodicity is:

```bash
0,15,30,45 * * * *
```

Yes, I describe it as a cron job pattern. Actually, the issue only happens at a fixed time — minutes 0, 15, 30, 45 every hour.

On a pgpool host which is deployed by bosh, there is the only one cron job, which is ntp updating, and the schedule is just defined at minutes 0, 15, 30, 45 every hour. So I stopped ntp updating cron job on pgpool hosts, however, unfortunately, the issue still came out.

Another observation is that this issue only appears on `pgpool_1` (`10.8.52.191`). Every time when the issue happened, my SSH connection to `pgpool_1` would throw a "broken pipe" error and be killed, while the connection to `pgpool_0` and other bosh deployed VMs did not have this problem.

Until this moment, we did not try to isolate the issue and did not narrow it to a particular component. Before doing this, let's do some simple analysis to see how many components would cause this issue.

- pgpool and watchdog

  From so far observation, we knew the issue only happened on pgpool host, did pgpool cause this issue?

- operating system

  If pgpool was not the reason, how about operating system, something wrong with the new stemcell? Actually, we upgraded bosh stemcell few days ago.


- network

  So far I can make sure there was no network break in lab, but how about other devices in network, was there IP conflict occured in network? This is just a wild guess and not any theory.

First, I changed the pgpool's configuration file to turn off watchdog, and restart pgpool job to take effect. Few minutes later, my SSH connection to `pgpool_1` throw "broken pipe" error again. Because there was no watchdog running on the host, there was no virtual IP address and split-brain syndrome would not be present.

Next, I stopped pgpool job on each host, my SSH connection to pool\_1 was killed again. So pgpool and watchdog were irrelevant to this issue, maybe?

But next experiment showed that pgpool was related actually. I brought up pgpool and watchdog on two hosts, and then used command `bosh stop pgoool 0 --hard` to delete the host `pgpool_0`. After doing that, the issue went away! I did other experiments trying to find out a underlying pattern of effect of pgpool:

- only stop pgpool job on `pgpool_0`, the issue still happened.
- only stop pgpool job on `pgpool_1`, the issue still happened.
- delete `pgpool_0` host, the issue went away, then stop pgpool job on `pgpool_1`, the issue came back.
- delete `pgpool_0` host, the issue went away, then brought up `pgpool_0`, the issue came back.
- only deployed one pgpool node with IP address `10.8.52.191`, the issue still happened.
- only deployed one pgpool node with IP address `10.8.52.190`, there was no any issue.
- deployed two dummy bosh VMs replacing pgpool VMs with the same IP addresses, the issue happened.

To determine whether it was a operating system issue, I opened `/var/log/kernel.log` file and saw,

```
/var/log/kernal.log @ pgpool_1
Jun 23 19:15:25 localhost kernel: [ 2416.770092] EXT4-fs (sdb2): re-mounted. Opts: (null)
Jun 23 19:15:25 localhost kernel: [ 2416.802015] EXT4-fs (sdb2): re-mounted. Opts: (null)
Jun 23 19:15:25 localhost kernel: [ 2416.807128] EXT4-fs (sdb2): re-mounted. Opts: (null)
Jun 23 19:30:39 localhost kernel: [ 3331.608756] EXT4-fs (sdb2): re-mounted. Opts: (null)
Jun 23 19:30:39 localhost kernel: [ 3331.633461] EXT4-fs (sdb2): re-mounted. Opts: (null)
Jun 23 19:30:39 localhost kernel: [ 3331.636631] EXT4-fs (sdb2): re-mounted. Opts: (null)
Jun 23 19:46:14 localhost kernel: [ 4266.522293] EXT4-fs (sdb2): re-mounted. Opts: (null)
Jun 23 19:46:14 localhost kernel: [ 4266.551129] EXT4-fs (sdb2): re-mounted. Opts: (null)
Jun 23 19:46:14 localhost kernel: [ 4266.554778] EXT4-fs (sdb2): re-mounted. Opts: (null)
Jun 23 20:00:33 localhost kernel: [ 5124.747828] EXT4-fs (sdb2): re-mounted. Opts: (null)
Jun 23 20:00:33 localhost kernel: [ 5124.772672] EXT4-fs (sdb2): re-mounted. Opts: (null)
Jun 23 20:00:33 localhost kernel: [ 5124.775212] EXT4-fs (sdb2): re-mounted. Opts: (null)
Jun 23 20:15:41 localhost kernel: [ 6032.961794] EXT4-fs (sdb2): re-mounted. Opts: (null)
Jun 23 20:15:41 localhost kernel: [ 6032.990478] EXT4-fs (sdb2): re-mounted. Opts: (null)
Jun 23 20:15:41 localhost kernel: [ 6032.994323] EXT4-fs (sdb2): re-mounted. Opts: (null)
Jun 23 20:30:49 localhost kernel: [ 6941.197722] EXT4-fs (sdb2): re-mounted. Opts: (null)
Jun 23 20:30:49 localhost kernel: [ 6941.221386] EXT4-fs (sdb2): re-mounted. Opts: (null)
Jun 23 20:30:49 localhost kernel: [ 6941.224402] EXT4-fs (sdb2): re-mounted. Opts: (null)
```

Mount point `sdb2` was re-mounted at `19:15`, `19:30`, `19:46`, `20:00`…, and network partitions were happening at those moments too. In a bosh deployed VM, `sdb2` is mounted by `bosh_agent`, so I checked `bosh_agent`'s log file:

```
[main] 2017/06/23 19:15:37 ERROR - App run Running agent: Sending heartbeat: disconnected
[main] 2017/06/23 19:15:37 ERROR - Agent exited with error: Running agent: Sending heartbeat: disconnected
[main] 2017/06/23 19:30:50 ERROR - App run Running agent: Sending heartbeat: disconnected
[main] 2017/06/23 19:30:50 ERROR - Agent exited with error: Running agent: Sending heartbeat: disconnected
[main] 2017/06/23 19:46:01 ERROR - App run Running agent: Sending heartbeat: disconnected
[main] 2017/06/23 19:46:01 ERROR - Agent exited with error: Running agent: Sending heartbeat: disconnected
[main] 2017/06/23 20:00:21 ERROR - App run Running agent: Sending heartbeat: disconnected
[main] 2017/06/23 20:00:21 ERROR - Agent exited with error: Running agent: Sending heartbeat: disconnected
[main] 2017/06/23 20:15:29 ERROR - App run Running agent: Sending heartbeat: disconnected
[main] 2017/06/23 20:15:29 ERROR - Agent exited with error: Running agent: Sending heartbeat: disconnected
[main] 2017/06/23 20:30:38 ERROR - App run Running agent: Sending heartbeat: disconnected
[main] 2017/06/23 20:30:38 ERROR - Agent exited with error: Running agent: Sending heartbeat: disconnected
```

Well, it turns out that the network partition happened on infrastrucure level and also affected bosh — when it was occurring, the connection between `bosh_agent` and `bosh_director` would be broken, and `bosh_agent` would was going to restart itself to fix this problem, that caused `sdb2` re-mounted.

Was it because IP conflict? I shut down the host `pgpool_1`, and then tried to ping IP address `10.8.52.191` which should belongs to `pgpool_1`, got some timeout, it's not pingable; I tried to SSH to that IP address later, got a failure as well. This test at least verified that there was no IP conflict at that moment.

Furthermore, I changed `pgpool_1`'s IP address from `10.8.52.191` to `10.8.52.192`, the issue could not be present in any way.

###### Day 2 summary:

- The issue only came out at fixed time — 0, 15, 30, 45 minutes every hour.
- It seems that pgpool is not relevant to the issue, but _somehow_ if `pgpool_0` was deleted while pgpool job running on `pgpool_1` could prevent this issue.
- The issue should present on infrastructure level, because bosh was affected.
- The issue should be related to IP address `10.8.52.191`, but I could not reach another host with this IP address.


#### Day 3

I talked with @Eng_B about this issue in the morning, @Eng_C happend to hear that the problem was related to the IP address `10.8.52.191`, he came up to us and said, this IP address he was using to deploy `bosh_init` on DR_WEST environment, and bosh reported an error when it was trying to create VM, because he had another important task, he left the VM on that error.

We went to DR_WEST vCenter and shutdown that `bosh_init` VM, the issue on `pgpool_1` went away. Next, I brought up the `bosh_init` VM again, and stopped ntp updating cron job. Finally I tried to run the ntp updating script manually on the `bosh_init` VM. As expected, this operation would trigger network partitioning on `pgpool_1`.

We're using bosh to orchestrate VMs on vCenter, which means each `bosh_director` is a hyperviser manager, it could allocate the resources in the hyperviser which it's managing. In this case, if we provisioned a VM with the same IP address as `pgpool_1` through promo environment's `bosh_director`, the issue could be avoided, because `bosh_director` would detect which IP address is in use, and report an error to user. However, we were trying to deploy another `bosh_director`(`bosh_init` contains a `bosh_director`) in DR_WEST, and there was an IP address overlap (`10.8.52.191`)  between these two `bosh_directors`, bosh is not able to do resources check cross hypervises.

At network side, we manage network with VLAN, each VLAN has a vSwitch, vSwitch is a network switch emulator at Layer 2. There is a ARP table cached in vSwitch, every time vSwitch is trying to connect two devices that have never been connected before, vSwitch will update the IP/MAC mapping in ARP table for these two devices. On `pgpool_1`, when `bosh_agent` set up a stream socket connection (TCP) to `bosh_director`, vSwitch will register IP/MAC mappings for `pgpool_1` and `bosh_director` into its ARP table. Few minutes later, while `bosh_init` is trying to do the ntp updating (UDP), vSwitch will update the IP/MAC mapping from "`10.8.52.191` \- `pgpool_1`\_mac\_addr" to "`10.8.52.191` \- `bosh_init`\_mac\_addr". At this moment, if `bosh_director` sends a message to `pgpool_1`, vSwitch will forward the message to `bosh_init`, and `bosh_init` will certainly refuse the message, so the stream socket connection between `bosh_director` and `pgpool_1` is terminated at `bosh_director` side. `bosh_agent` sends heartbeat to `bosh_director` every 5 seconds, so few seconds later, `bosh_agent` on `pgpool_1` sends a heartbeat over it stream socket connection, it will find the connection has been terminated, so `bosh_agent` on `pgpool_1` will try to restart itself to fix this issue. When `bosh_agent` is trying to setup a new stream socket connection over TCP, vSwtich will update the IP/MAC mapping from "10.8.52.191 \- `bosh_init`\_mac\_addr" to "10.8.52.191 \- `pgpool_1`\_mac\_addr", which means the network on `pgpool_1` is recovered.

This could explain most phenomenons we observed before, except one: why pgpool job on `pgpool_1` could prevent network partitioning only if `pgpool_0` was deleted?

I checked process list on `pgpool_1`, and found if I deleted `pgpool_0`, the watchdog on `pgpool_1` had been still sending heartbeat to `pgpool_0`'s IP address at 2-second interval. If I reduce the freqency of heartbeat sending from watchdog, such as into 10 seconds, then the network partition could be present on `pgpool_1` even if I deleted `pgpool_0`.

vSwitch only update its ARP table when it's trying to connect two devices that have never been connected before. If `pgpool_0` is alive, `10.8.52.190` is reachable, when watchdog on `pgpool_1` is establishing datagram socket connection (UDP) to `pgpool_0`, vSwitch will update IP/MAC mappings in ARP table for both host. But once the two hosts have been connected, vSwitch will not update their IP/MAC mappings until one of these two hosts changes its IP or MAC address. However, if `pgpool_0` is deleted, `10.8.52.190` is unreachable, at this moment, vSwitch thinks the IP address `10.8.52.190` is a new device in network, it will update IP/MAC mapping for `pgpool_1` and do `arping` to find out MAC address of `10.8.52.190`. No matter it could find out the MAC address of `10.8.52.190`. The point is when the heartbeats are sent from `pgpool_1` while `pgpool_0` is unreachable, vSwitch updates IP/MAC mapping for `pgpool_1`.

Heartbeats from watchdog is sent every 2 seconds, but heartbeats from `bosh_agent` is sent every 5 seconds, so a higher frequency heartbeat from watchdog makes stream socket connection between `bosh_agent` and `bosh_director` would not be terminated, and the network partition could not be observed.

###### Day 3 summary

- IP conflict is the root cause of this issue.
- NTP updating on `bosh_init` DR_WEST triggers network partitioning on `pgpool_1`.
- Higher frequency heartbeats from watchdog makes network partitioning could not be observed on `pgpool_1`.

## Treatment & Experience 

Since we have figured out the root cause of this issue, the treatment is pretty simple — changes the IP address of `bosh_init` in DR_WEST. But this issue leaves us some thoughts.

- When we were deploying postgres service on promo environment, we did not ask for IP addresses from NetOps team. We just randomly ping some IP addresses, if an address was unreachable, we adopted it. We chose a wrong way to get IP address, but the underlying reason pushing us to choose this way is, the cost of gaining IP addresses is high for engineers. If we need an IP address, we have to send an email request (or open a ticket) to NetOps team to ask for it, and then wait for couple hours (or couple days). 

- In our current deployment manifests, we specify IP addresses for each VM manually. It causes that every time we need add a new deployment, we have to define network section in manifest very carefully and make sure bosh would not allocate an in-use IP address for compiling VM. All these works are done by manpower, it's difficult to avoid making mistakes for human being. 

Imagine, if we have a centralized system to manage IP addresses (and other network resources), NetOps just need to define a subnet which could be free to use, all engineers can login to this system to get IP addresses by themselves, I think nobody would use that wrong way to get IP address. People love self-service because of independency.

And, actually, bosh offers a feature what could automatically allocate IP addresses, we just need to specify a subnet definition in network section, the rest of things you can leave to bosh to complete.  

IP address is an important resource at infrastructure level, and we should manage them carefully and intelligently.