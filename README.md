## Introduction
running repairs can be challenging, this is a simple utility that allows you to run sub-range repairs on a supplied list of tables and will pace the repair to complete within a supplied number of days. The tool leverages 'ssh' and requires ssh access to all nodes in the cluster. The tool can be configured to adjust the rate of repair to complete within the given repair time, in addition the progress of the repairs is saved as the repairs progress allowing the command to be stopped and restarted.


## Instructions
Clone the repo and copy to the single server which can access all nodes in the cluster via ssh.
```
git clone https://github.com/gregory-smith/repairtool.git
```
The tool requires the token ranges of all nodes in the cluster to build a tasklist of repairs that need to be performed. Execute the following command save the output
```
nodetool ring
```
Once a tasklist has been created, the tool will use this file exclusively to run repairs. If the topology changes, simply remove the tasklist file and restart the repairtool. When the repairtool starts it will create a new tasklist if one does not exist.

Next, configure the repairtool to execute repairs. Sample configurations can be found in the config folder.

```
{
  "config": {
    "repair_time": 10,
    "pace_type": "fixed",
    "pace_factor": 1.0,
    "keyspace": "OpsCenter",
    "ssh_user": "ubuntu",
    "nodetool_command": "/usr/share/dse-4.8.14/resources/cassandra/bin/nodetool "
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
* keyspace - keyspace containing the tables requiring repair
* tables - list of tables to repair, the repairtool will execute repairs in parallel against all tables on a given line. If you want to repair multiple tables, ensure that the tables are space delimited.
* pace_factor - see description of pace_type below
* pace_type - Either "Fixed" or "Dynamic".
A "Fixed" pace_type will, pause between each sub-range repair for a period equal to the time of the previous repair multipled by the pace_factor. For example, if the previous repair completed in 30 seconds and the pace_factor = 2.0, then the repairtool will pause for 30*2.0 = 60 seconds before starting the next repair.
A "Dynamic" pace_type, extends this functionality and will periodically, calculate if the repairs are proceding at the correct rate to complete within the given time, and adjust the "pace_factor" accordingly.


Finally start the repairtool

```
Usage: repairtool <ring information> <configfile> <tasklist>

./repairtool ring config/bigtable.json tasklists/tasklist.bigtable
```


Sample output of repairtool creating a new tasklist

```
./repairtool ring config/bigtable.json tasklists/tasklist.bigtable
INFO Parsing configuration options
INFO Reading Cluster topology
Reading topology : ring
INFO Generating NEW repair tasklist
INFO Creating task list
INFO Table sets : 1
INFO Token Ranges : 3200
INFO Generated 3200 tasks
INFO Executing repairs using task list : tasklists/tasklist.bigtable
INFO Repair time : 1 Days (Total Tasks : 3200 )
INFO Commencing repair cycle in 20 seconds
INFO Checking progress at 2017-10-04 09:46:21
INFO Repair cycle started : 2017-10-04 09:46:21
.
.


```

Sample output of repairtool using an existing tasklist

```
./repairtool ring config/bigtable.json tasklists/tasklist.bigtable
INFO Parsing configuration options
INFO Reading tasklist for repairs
INFO Executing repairs using task list : tasklists/tasklist.bigtable
INFO Repair time : 1 Days (Total Tasks : 3200 )
INFO Commencing repair cycle in 20 seconds
INFO Checking progress at 2017-10-04 09:48:34
INFO Repair cycle started : 2017-10-04 09:48:34
.
.

```

Periodically (every 1% of tasks complete), the repairtool will display the current state of the repair and if dynamic pacing has been enabled, indicate if it will be speeding up or slowing down the speed of the repair process.

```
INFO Checking progress at 2017-10-04 09:52:13
INFO Repair cycle started : 2017-10-03 15:35:55
INFO Repair Task percentage complete : 14.0
INFO Repair Task percentage time used : 3.80660345724
INFO Dynamic pacing enabled, Reducing speed of repairs (38.443359375)
```
