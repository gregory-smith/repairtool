## Introduction
running repairs can be challenging, this is a simple utility that allows you to run sub-range repairs on a supplied list of tables or an entire keyspace. The repairtool will pace the repair to complete within a supplied number of days. The tool leverages 'ssh' and requires ssh access to all nodes in the cluster. The tool can be configured to adjust the rate of repair to complete within the given repair time, in addition the progress of the repairs is saved as the repairs progress allowing the command to be stopped and restarted.

It is possible to run multiple repairtool's against a single cluster by creating splitting the repair into multiple tasks using the repair_threads configuration parameter. In addition, by specifying start and end times, start_time>end_time it is possible to specify certain times of day when the repair process with run. Simply set the start_window and end_window to the same value to disable this feature.


## Instructions
Clone the repo and copy to the single server which can access all nodes in the cluster via ssh.
```
git clone https://github.com/gregory-smith/repairtool.git
```
The tool requires the token ranges of all nodes in the cluster to build a tasklist of repairs that need to be performed. Execute the following command on your cluster and save the output.
```
nodetool ring
```
Once a tasklist has been created, the tool will use this file exclusively to run repairs. If the topology changes, simply remove the tasklist file(s) and generate a new set using the 'repairtool generate' command.

Next, configure the repairtool to execute repairs. Sample configurations can be found in the config folder.

```
{
  "config": {
    "repair_time": 10,
    "start_window": "17:00",
    "end_window":"08:00",
    "repair_threads":1,
    "pace_type": "Dynamic",
    "pace_factor": 1.0,
    "keyspace": "OpsCenter",
    "full_keyspace": false,
    "ssh_user": " ubuntu",
    "nodetool_command": "/usr/share/dse-5.0.11/resources/cassandra/bin/nodetool "
  },
  "tables": [
    "rollups60 rollups7200 rollup_state",
    "events"
  ]
}
```

* ssh_user - ssh account to use to connect to nodes in the cluster
* nodetool_command - full path to the nodetool command
* repair_time - Time to complete the repair in days
* repair_threads - split the repair jobs into multiple tasklists to execute repair commands in parallel.
* start_window/end_window: Specify a time window in 24 hour clock notation when repairs are allowed to execute.
* keyspace - keyspace containing the tables requiring repair
* full_keyspace - True/False flag to indicate if you want the entire keyspace repaired or individual listed tables
* tables - list of tables to repair, the repairtool will execute repairs in parallel against all tables on a given line. If you want to repair multiple tables, ensure that the tables are space delimited.
* pace_factor - see description of pace_type below
* pace_type - Either "Fixed" or "Dynamic".
A "Fixed" pace_type will, pause between each sub-range repair for a period equal to the time of the previous repair multipled by the pace_factor. For example, if the previous repair completed in 30 seconds and the pace_factor = 2.0, then the repairtool will pause for 30*2.0 = 60 seconds before starting the next repair.
A "Dynamic" pace_type, extends this functionality and will periodically, calculate if the repairs are proceding at the correct rate to complete within the given time, and adjust the "pace_factor" accordingly.

Next generate your tasklists.
Sample output of repairtool creating a new set of 10 tasklists

```
./repairtool generate test-cluster config/keyspace.json tasklists/keyspace.tsk-
INFO Parsing configuration options
INFO Reading Cluster topology
Reading topology : test-cluster
INFO Generating NEW repair tasklist
INFO Creating task list
INFO Table sets : 2
INFO Full keyspace repair : True
INFO Threads : 10
INFO Token Ranges : 3870
INFO Generated 3870 tasks
Saving file : tasklists/keyspace.tsk-0
Saving file : tasklists/keyspace.tsk-1
Saving file : tasklists/keyspace.tsk-2
Saving file : tasklists/keyspace.tsk-3
Saving file : tasklists/keyspace.tsk-4
Saving file : tasklists/keyspace.tsk-5
Saving file : tasklists/keyspace.tsk-6
Saving file : tasklists/keyspace.tsk-7
Saving file : tasklists/keyspace.tsk-8
Saving file : tasklists/keyspace.tsk-9
```

Finally, supply a tasklist to the repairtool and start repairing that cluster.

Sample output of repairtool using an existing tasklist

```
./repairtool execute tasklists/keyspace.tsk-0
INFO Parsing configuration options
INFO Reading tasklist for repairs
INFO Executing repairs using task list : tasklists/keyspace.tsk-0
INFO Repair time : 2 Days (Total Tasks : 82 )
INFO Commencing repair cycle in 20 seconds
INFO Checking progress at 2018-04-26 16:32:43
INFO Repair cycle started : 2018-04-26 16:25:54
INFO Repair Task percentage complete : 62.0
INFO Repair Task percentage time used : 0.237121653916
[2018-04-26 20:32:47,054] Starting repair command #234, repairing keyspace keyspace1 with repair options (parallelism: parallel, primary range: false, incremental: false, job threads: 1, ColumnFamilies: [], dataCenters: [], hosts: [], runAntiCompaction: false, # of ranges: 1)
[2018-04-26 20:32:47,136] Repair session fb0ef5a0-4990-11e8-925d-df7ecc96cb72 for range [(-1465814810869877305,-1463958783234582035]] finished (progress: 100%)
[2018-04-26 20:32:47,142] Repair completed successfully
.
.

```

Periodically (every 1% of tasks complete), the repairtool will display the current state of the repair and if dynamic pacing has been enabled, indicate if it will be speeding up or slowing down the speed of the repair process to meet the configured repair time.

```
INFO Checking progress at 2017-10-04 09:52:13
INFO Repair cycle started : 2017-10-03 15:35:55
INFO Repair Task percentage complete : 14.0
INFO Repair Task percentage time used : 3.80660345724
INFO Dynamic pacing enabled, Reducing speed of repairs (38.443359375)
```
