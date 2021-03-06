# MariaDB OPS Guide

This document will cover the following topics and is intended as a guide for regular MariaDB DBA tasks such as backup, restore, rebuilding slave nodes, setting up new master etc. Assumption is RHEL/CENTOS using MariaDB enterprise server & Backup.

- Prerequisites
- Full Backup
- Full Restore
  - Rebuilding a Fresh Slave from a MariaBackup
- Promoting a Slave node to a Master

## Prerequisites

To be able to use **`mariabackup`** we need to do the following
- Create a dedicated DB user with grants required to run MariaBackup
- Set UP `ulimit` on the OS for the OS user responsible to execute `mariabackup`

### Create MariaBackup user

Assuming that the backup MariaDB user account is `mariabackup@localhost` or any specific user that you might have already available.

```sql
CREATE USER mariabackup@localhost IDENTIFIED BY '<password>';
GRANT RELOAD, PROCESS, LOCK TABLES, REPLICATION CLIENT ON *.* TO mariabackup@localhost;
```

The password is your own choice following the password policy set within the organization.

### Set `ulimit`

Check the current limits for your current user, **`root`**, in this case.

```bash
shell> ulimit -Sn
1024

shell> ulimit -Hn
4096
```

This means, there is a soft/hard limit of 1024/4096 files opened at any given time. This might be too low for a larger database and needs to be increased.

Assuming, the MariaDB Backup is going to be execute using the **`mysql`** OS user, for instance.

Using the OS **`root`** user, execute the following statements to set `ulimit` for `mysql` OS user. This will define the soft/hard limits for the `mysql` user to 65535 in the file `/etc/security/limits.conf`. 

```bash
shell> echo "mysql              soft    nofile          65535" >> /etc/security/limits.conf
shell> echo "mysql              hard    nofile          65535" >> /etc/security/limits.conf
```

Once this is done, you can cat the `/etc/security/limits.conf` file to get the following output

```txt
shell> grep mysql /etc/security/limits.conf
mysql              soft    nofile          65535
mysql              hard    nofile          65535
```

This can be further verified to confirm if the limits are in place with the help of `ulimit -Sn` to check the soft limit and `ulimit -Hn` to check the hard limit values. 

```bash
shell> ulimit -Sn
65535

shell> ulimit -Hn
65535
```

Make sure that this is done for the correct user, the user that executes **`mariabackup`**

## Full Backup

Following is a sample script that takes performs a full backup, copy the contents of the block in `mariadb_full_backup.sh` and grant read/execute permissions 

- `chmod 750 mariadb_full_backup.sh`. 

The user executing the script must have permissions on the MariaDB data directories and other folders that are being used here.

```bash
#!/bin/bash

. /glide_backup/backup/.mysqlbackup_info

dt=`date +%Y-%m-%d_%H-%M-%S`
BASE_DIR=/glide_backup/backup
BACKUP_DIR=${BASE_DIR}/MARIABACKUP
TARGET_DIR=${BACKUP_DIR}/${dt}

# If the Log folder does not exists, create it
if [ ! -d ${BASE_DIR}/logs ]
then
   mkdir -p ${BASE_DIR}/logs
fi

Log_Location=${BASE_DIR}/logs/MARIADB_Backup_${dt}.log

#Initialize the Log File
echo "Started: $(date)" > ${Log_Location}
echo "" >>  ${Log_Location}

UserName=$INFO1
Password=$INFO2

MariaDBStatus=$(systemctl status mariadb | grep "active (running)" | wc -l)

if [ ${MariaDBStatus} -ne 1 ]
then
  echo "MariaDB not available $(systemctl status mariadb | grep "Active:"), ABORT!" >> ${Log_Location}
  exit 1
fi

# Check if path already exists or not
if [ ! -d ${TARGET_DIR} ]
then
  mkdir -p ${TARGET_DIR}
else
  echo "Backup target directory ${TARGET_DIR} already exists, ABORT!" >> ${Log_Location}
  exit 1
fi

# Execute MariaBackup and check the return status
mariabackup --backup --target-dir=${TARGET_DIR} --user=${UserName} --password=${Password} >> ${Log_Location} 2>> ${Log_Location}
retStatus=$?

# Check if Backup Successful then execute prepare stage
if [ ${retStatus} -eq 0 ]; then
        # Backup successful, find backup directory
        echo "===================================================">> ${Log_Location}
        echo "MariaDB Backup is sucessfully completed: `date`">> ${Log_Location}
        echo "Executing prepare stage now: `date`">> ${Log_Location}     
        echo "===================================================">> ${Log_Location}
        echo "" >> ${Log_Location}
        mariabackup --prepare --target-dir=${TARGET_DIR} >> ${Log_Location} 2>> ${Log_Location}
        retStatus=$?
fi

SuccessfulCompletion=$(cat ${Log_Location} | grep "completed OK!" | wc -l)

# Check if Prepare or Backup failed
if [[ ${retStatus} -ne 0 || ${SuccessfulCompletion} -ne 2 ]]; then
        echo "======================================================================================================"
        echo "MariaDB Backup FAILED!, Please check ${Log_Location} for details: `date`"
        echo "======================================================================================================"
        echo ""
        exit 1
fi
echo ""
echo "MariaDB Backup completed successfully!"
echo "Check the ${Log_Location} for more details"
echo ""

echo "========================" >> ${Log_Location}
echo "Backup BEFORE deletion  " >> ${Log_Location}
echo "========================" >> ${Log_Location}
echo "===============================================================" >> ${Log_Location}
ls -ltr ${BACKUP_DIR} >> ${Log_Location}
echo "===============================================================" >> ${Log_Location}

echo "Number of file(s) deleted `date` : " `find  ${BACKUP_DIR} -type d -mtime +3` >> ${Log_Location}
find ${BACKUP_DIR} -type d -mtime +3 -exec rm -rf {} \; >> ${Log_Location}
echo "">> ${Log_Location}
echo "========================" >> ${Log_Location}
echo "Backup AFTER deletion   " >> ${Log_Location}
echo "========================" >> ${Log_Location}
echo "===============================================================" >> ${Log_Location}
ls -ltr ${BACKUP_DIR}>> ${Log_Location}
echo "===============================================================">> ${Log_Location}
echo "">> ${Log_Location}
exit 0
```

Important things to take note in this script

- **`/glide_backup/backup/.mysqlbackup_info`**
  - This file contains user name and the password as a hidden file.
  - The file contents are as follows, the `user (INFO1)` and `password (INFO2)` are based on the backup user that has the necessary grants required to run MariaDB Enterprise Backup.
    ```
    INFO1=mariabackup
    INFO2=SecretPassword!23
    ```
  - These values are imported at the begining of the script and used by te script when calling MariaBackup commandline.
  - Make sure to keep the file's ownership with very limited and just read only permissions to the OS user doing the backup while the other users, NO permission. 
    - `chown mysql:mysql /glide_backup/backup/.mysqlbackup_info`
      - File ownership changed to `mysql:mysql` user/group assuming that `mysql` user is being used to run the backup script.
    - `chmod 400 /glide_backup/backup/.mysqlbackup_info`
      - Just read only permission to `mysql` user while no access to others or group.
- **`BASE_DIR=/glide_backup/backup`**
  - Is the home directory where all the backups and logs will be stored
- **`BACKUP_DIR=${BASE_DIR}/MARIABACKUP`**
  - Is the target folder where the actual backups be stored in dedicated sub folders.
- **`TARGET_DIR=${BACKUP_DIR}/${dt}`**
  - Under the `BACKUP_DIR` this folder will be automatically created with `Year-Month-Day_Hour-Min-Sec` format to contain the actual daily backups.
- **`Log_Location=${BASE_DIR}/logs/MARIADB_Backup_${dt}.log`**
  - Log files for each backup with the same prefix `Year-Month-Day_Hour-Min-Sec`
  - This log file will indicate if the backup was successful or failed. 
    - `completed OK!` will be written twice, once for the `--backup` and once  for the `--prepare`

Once the backup is done successfully, `--prepare` is automatically executed. A final status check to ensure the backup and prepare are successful will lead to deleting backup folders older than 3 days. This means, at any given time, only 3 successfully completed backups will be available on the server.

## Full Restore

Based on the backup script, we know that the backup was created in a dynamic folder under `${TARGET_DIR}` The restore will be similar.

Copy the contents of the following block in to a shell script `mariadb_restore.sh` and grant read/execute permission. The OS user executing the script must have permissions to read/write the MariaDB data directory structures and other folders referred in the script

- `chmod 750 mariadb_restore.sh`

```bash
#!/bin/bash

RestorePoint=$1

dt=`date +%Y-%m-%d_%H-%M-%S`

BASE_DIR=/glide_backup/backup
BACKUP_DIR=${BASE_DIR}/MARIABACKUP
TARGET_DIR=${BACKUP_DIR}/${RestorePoint}
Log_Location=${BASE_DIR}/logs/MARIADB_Restore_${RestorePoint}.log

BINLOG_DIR=/glide/mysql/binlog
DATA_DIR=/glide/mysql/data

if [ "$#" -ne 1 ]; then
    echo "Illegal number of arguments to the script..."
    echo ""
    echo "Syntax:"
    echo "       shell> ./mariadb_restore.sh <Backup Folder>"
    echo "       shell> ./mariadb_restore.sh 2021-01-01_03-00-00"
    echo ""
    echo "The above folder '2021-01-01_03-00-00' must exist under ${BACKUP_DIR}"
    echo "Restore logs will be generated under ${Log_Location}"
    echo ""
    exit 1
fi

MariaDBStatus=$(systemctl status mariadb | grep "inactive (dead)" | wc -l)

if [ ${MariaDBStatus} -ne 1 ]
then
  echo "MariaDB service not stopped, Stop the service before restoring, ABORT!"
  exit 1
fi

# Check if target directory does not exists
if [ ! -d ${TARGET_DIR} ]
then
  echo "Backup target directory ${TARGET_DIR} does not exists, ABORT!"
  exit 1
fi

if [ ! -d ${DATA_DIR} ]
then
  echo "Data directory ${DATA_DIR} does not exists, ABORT!"
  exit 1
fi

if [ ! -d ${BINLOG_DIR} ]
then
  echo "Binlog directory ${BINLOG_DIR} does not exists, ABORT!"
  exit 1
fi

#Initialize the Log File
echo "Started: $(date)" > ${Log_Location}
echo "" >> ${Log_Location}

echo "Backup Configuration" >> ${Log_Location}
echo "-----------------------------------------------------------" >> ${Log_Location}
echo "Backup Base Directory...: ${BACKUP_DIR}" >> ${Log_Location}
echo "Backup Target Directory.: ${TARGET_DIR}" >> ${Log_Location}
echo "BinLog Directory........: ${BINLOG_DIR}" >> ${Log_Location}
echo "Restore Data Directory..: ${DATA_DIR}" >> ${Log_Location}
echo "-----------------------------------------------------------" >> ${Log_Location}
echo "" >> ${Log_Location}

echo "Erasing Binlogs directory ${BINLOG_DIR}"  >> ${Log_Location}
rm -rf ${BINLOG_DIR}/*
eraseBinlogStatus=$?

echo "Erasing Data directory ${DATA_DIR}"  >> ${Log_Location}
rm -rf ${DATA_DIR}/*
eraseDataDirStatus=$?

if [[ eraseDataDirStatus -ne 0 || eraseBinlogStatus -ne 0 ]]
then
   echo "Failed to erase data directory / binary logs folder, Aborting!" >> ${Log_Location}
fi

echo ""
echo "Restoring MariaDB backup" >> ${Log_Location}


# Execute MariaBackup and check the return status
mariabackup --copy-back --target-dir=${TARGET_DIR} --datadir=${DATA_DIR} >> ${Log_Location} 2>> ${Log_Location}
retStatus=$?

# Check if Backup Successful then execute prepare stage
if [ ${retStatus} -ne 0 ]; then
    echo "MariaDB Backup FAILED!, Please check ${Log_Location} for details"
    echo "===================================================">> ${Log_Location}
    echo "MariaDB Backup FAILED!, Please check ${Log_Location} for details: `date`" >> ${Log_Location}
    echo "===================================================">> ${Log_Location}
    echo ""
    exit 1
fi

echo "Changing Ownership of the Data Directory ${DATA_DIR}" >>  ${Log_Location}
chown -R mysql:mysql ${DATA_DIR}
retStatus=$?

if [ ${retStatus} -ne 0 ]; then
    echo "Failed to change ownership of the Data Directory ${DATA_DIR} using $(whoami)"
    exit 1
fi

echo ""
# Backup Restore successful, find backup directory
echo "MariaDB Backup Restore successful!, Please check ${Log_Location} for more details"

echo "===================================================">> ${Log_Location}
echo "MariaDB Backup Restore is sucessfully completed: `date`">> ${Log_Location}
echo "===================================================">> ${Log_Location}
echo "" >> ${Log_Location}
echo ""
exit 0
```

Important points about the restore script

- MariaDB service must be stopped before triggering the script
- Data directory is hardcoded to **`DATA_DIR=/glide/mysql/data`** change if needed
  - This folder will be automatically be cleaned up by the script
- Binary Logs directory is hardcoded to **`BINLOG_DIR=/glide/mysql/binlog`** change if needed
  - This folder will be automatically be cleaned up by the script
- Backup Directory a.k.a Target Directory is a mix of the following three
  - **`BASE_DIR=/glide_backup/backup`**
  - **`BACKUP_DIR=${BASE_DIR}/MARIABACKUP`**
  - **`TARGET_DIR=${BACKUP_DIR}/${RestorePoint}`**
    - **`${RestorePoint}`** is the user input parameter, it's the final folder name that contains the actual backup required to be restored
  - Change any of the above path as needed, however the **`RestorePoint`** is a user input passed in to the script as a command line argument.
- After successful restore, the ownership of the Data Directory **`/glide/mysql/data`** will be automatically be changed to **`mysql:mysql`**
- Restore logs will be found under **`Log_Location=${BASE_DIR}/logs/MARIADB_Restore_${RestorePoint}.log`**, change if needed.
As an example of the restoration, there are three backup folders under **`/glide_backup/backup/MARIABACKUP/`**, we can use any of those folders as an argument to the restore script and that database will be restored.

As an example of the folder structure and how to execute the restore script. Let's say we want to restore the backup under `2021-01-18_08-38-11` folder.

```txt
shell> systemctl stop mariadb

shell> cd /glide_backup/backup/MARIABACKUP/
shell>  ls -rtl
total 0
drwxr-xr-x. 2 root root   6 Jan 18 08:35 2021-01-18_08-35-27
drwxr-xr-x. 6 root root 250 Jan 18 08:36 2021-01-18_08-36-37
drwxr-xr-x. 6 root root 250 Jan 18 08:38 2021-01-18_08-38-11

shell> ./restore.sh 2021-01-18_08-38-11
```

### Rebuilding a Slave after Restore

Once the backup has been restored, the backup location (**TARGET_DIR**) contains a file named `xtrabackup_binlog_info` This file contains a single line of data, binlog file name, binlog posiiton and the **GTID**

```
shell> cat /glide_backup/backup/MYSQLBACKUP/2020-12-19_15-10-25/xtrabackup_binlog_info
mariadb-binlog.000003	4762	0-1000-20
```


The **`0-1000-20`** is the GTID position, this is important when setting up replication

To set this node as a slave to the existing Master, just need to follow the steps as per follows:

- `MariaDB> set global gtid_slave_pos='0-1000-20';`
- `MariaDB> CHANGE MASTER TO MASTER_HOST='<Master IP>',MASTER_PORT=<Master-Port>, MASTER_USER='<Master User>', MASTER_PASSWORD='User Password>', MASTER_USE_GTID=slave_pos;`
- `START SLAVE;`
- Check slave status using `show slave status\G` and ensure no errors, IO and SQL threads are running "YES"

This will let the slave start from the point of backup and pull all the new transactions from the transaction id following `"0-1000-20"` Where 0 is the domain ID, 1000 is the Master's server ID and 20 is the number of transaction that have happened so far when the backup was taken.

There is no need to reset master/slave at this time or the need to do `skip-slave-start`

## Promoting a Slave node to a Master Manually

Finally the process of manually promoting one of the slave nodes as the new Master

Assuming the current master is down and unrecoverable, we can decide to promote any of the existing nodes as the new master, the steps are as follows

- Identify the slave node with the highest GTID
  - Login to the slave nodes and check `show global variables like 'gtid_slave_pos';` 
  - The node with the highest numnber is the most up to date slave node and can be made the new master.
- On the selected slave node, execute the following commands to reset their slave status
  - **`reset slave all;`**
  - **`reset master;`**
    - Now this node is no longer a slave of the original master.
- Set it's read_only status to `off` 
  - **`set global read_only=off;`**
  - Make sure to set the same on the `/etc/my.cnf` file as well so that when the node is restarted, it will retain this config.
- Take note of the node's `master status`
  - `show master status;`
- On the remaining slave nodes, reset their slave and master status and execute `CHANGE MASTER` to point to the new master.
  - `reset master; reset slave all;`
  - `CHANGE MASTER TO MASTER HOST <New Master IP>, MASTER_PORT=<Master-Port>, MASTER_USER='<Master User>', MASTER_PASSWORD='User Password>', MASTER_LOG_FILE='<Master's current Binlog File'>,MASTER_LOG_POS=<Master's current position>; START SLAVE`
  - `SHOW SLAVE STATUS\G`
  - if everything is good, switch it to use GTID based replication
    - `STOP SLAVE; CHANGE MASTER TO MASTER_USE_GTID=slave_pos; START SLAVE;` 
    - `SHOW SLAVE STATUS\G` to verify the replication status.
- Repeat this on all the remaining slave nodes until all point to the new master.
- When the original master comes back online, this node most likely will not be able to join back the cluster and will need to be rebuild based on a fresh backup from the new master, process as explained previously **"Rebuilding a Slave after restore"**

***Note:** However on a controlled switch over to a new master, the original node should be able to join back the cluster as a slave without needing a rebuild.*

## Promoting a Slave as the new Master through MaxScale (Failover/Switchover)

To swith Master from one node to Another or even from one data center to another data center, the process is as follows:

- Stop all the application nodes, ensure no new transaction is coming on to the servers (GTID not moving)
- On the Active MaxScale node execute the following
  - `maxctrl call command mariadbmon switchover -t999 <MariaDB Monitor Name> <New Master Server Name> <Old MAster Server Name>` 
  - This will swithover from the old master to a new master.
  - To check, which is the active MaxScale, execute `maxctrl show maxscale | grep -i "passive"`
    - The node that returns **`"passive:false"`** is the active node
- Restart the application nodes and traffic will start to flow to the new Master automatically.

Another scenario is if the master node goes down while all the MaxScale nodes are running in Passive mode, we can manually trigger failover through MaxScale but first need to switch the MaxScale to "Active"

- `maxctrl alter maxscale passive false`
- `maxctrl call command mariadbmon failover <MariaDB Monitor Name>`

Other useful MaxScale commands are 

- `maxctrl call command mariadbmon rejoin <MariaDB Monitor Name> <Old Server Name>`
- `maxctrl call command mariadbmon reset-replication <MariaDB Monitor Name>`
- `maxctrl call command mariadbmon reset-replication <MariaDB Monitor Name> <New Master Name>`
- `maxctrl call command mariadbmon switchover <MariaDB Monitor Name>`
- `maxctrl call command mariadbmon switchover <MariaDB Monitor Name> <New Master Name>`

One rule to follow is to stop all the App nodes and switch Active MaxScale to Passive mode whenever any activity is done on the Master node in the cluster. For all the Slave nodes, it does no matter and business as usual. 

Always shutdown MariaDB gracefully before rebooting any server.

### Thank You
