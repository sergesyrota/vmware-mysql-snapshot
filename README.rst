Consistent MySQL snapshots with VMware file system quiescence
=====================

The goal of this script is to ensure consistent MySQL state at the time VMware snapshot is taken, while not creating too much of a problem to everything else that is happening on the server.

Requirements
-------------------------------------------
* You'll need a Linux OS under VMware (ESX 4 & 5 tested, not sure about compatibility with others)
* VMware tools installed and running
* Snapshots should be taken with "Quiesce guest file system"
* SUPER, RELOAD, PROCESS privileges to MySQL instance on localhost
* PHP's MySQLi extension (usually built-in)

Bonus
-------------------------------------------
As a bonus, script will dump master status into log file right after successful flush. This makes it extremely easy to use backup done with snapshot like that to resync your slave (or seed new slave).

Installation
-------------------------------------------

.. 001-commands-start
Place pre-freeze-script file in /usr/sbin, and run these::

	chmod 755 /usr/sbin/pre-freeze-script
	ln -s /usr/sbin/pre-freeze-script /usr/sbin/post-thaw-script
	ln -s /usr/sbin/pre-freeze-script /usr/sbin/pre-freeze-mysql-lock
.. 001-commands-end
	
This will:

1. Set proper execution permissions on the main file
2. Create a symlink for post-thaw action after VMware is done taking a snapshot
3. Create a symlink for a helper command that will be actually acquiring the snapshot

Then edit pre-freeze-script, typing in your MySQL credentials (you need SUPER privileges for this). You can also modify default values for run and log file, as well as timout values.

Add your log file to logrotate, to avoid its uncontrolled growth. Sample vmware-mysql-snapshot.logrotate file provided with default parameters for your convenience. Just copy it into /etc/logrotate.d folder.

How it works
-------------------------------------------
All files are symlinks to the same source, and depending which file is ran, appropriate function is invoked.

When VMware wants to take a snapshot, it invoked pre-freeze-script through vmware-tools. This function connects to MySQL, and starts looking for a good time to jump in. Definition of good time: there are no long running queries on the server (running $_config['lockWaitTimeout'] seconds or longer). At this point, queries in status "Sleep", "Delayed insert", "Binlog Dump", and "Connect" are ignored (list is configurable), as they don't block flush command. When it found the window of opportunity, it forks a child process calling pre-freeze-mysql-lock with nohup (so that it is not killed when this one is done), suppressing all output (so that it continues in the background without pausing).

Child process makes a new connection to MySQL and writes its status to the run file ($_config['runFile']). After that, it runs "FLUSH TABLES WITH READ LOCK" on MySQL instance. At this point, 2 things can happen:

1. If the flush is successful, we will get replication master position (if server is configured as such) for log file. Then write to the status file what happened and will just sit there waiting for everyone to do their thing. That means parent process will check the file, see that lock is aquired, and will exit with the status of 0. Then VMware will proceed with taking a file system snapshot.
2. We will happen to launch after some heavy query started, and we'll be sitting there and waiting when it finishes (what if it takes hours?). This can go on, but only for a limited time (approximately $_config['lockWaitTimeout'] seconds). If we are unable to flush tables during this time, our parent (knowing PID and MySQL connection from the runFile) will terminate this process, and kill MySQL query. This will free MySQL to accept new connections, and we'll wait for another time to take a snapshot.

After VMware is done taking it's snapshot, vmware-tools will call post-thaw-script. This will read current status form the run file, and will terminate blocking flush tables in MySQL, and kill pre-freeze-mysql-lock process. In case VMware takes too long, or something else happens, MySQL lock will be released after $_config['maxLockTime'] seconds.
