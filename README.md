# Database Week 6: Backup, PITR and Streaming Replication Lab

PostgreSQL 16 on Windows. The `bootcamp` database has a `students` table.

## Step 1: Logical backup

I backed up the `bootcamp` database with `pg_dump -Fc` and checked it with `pg_restore --list`. I restored it into `bootcamp_check`, and both databases had the same row count (17), so the backup is valid.

![Step 1 restore and row count](images/step_1.png)

## Step 2: WAL archiving and base backup

I enabled WAL archiving with `wal_level = replica`, `archive_mode = on` and an `archive_command` that copies WAL files to `C:\pgarchive`. I restarted PostgreSQL, forced a WAL switch with `pg_switch_wal()` to confirm files were archived, then took a base backup with `pg_basebackup`.

![Step 2 archive settings](images/step_2.png)
![Step 2 WAL files archived](images/step_3.png)
![Step 2 base backup](images/step_4.png)

## Step 3: Disaster and point-in-time recovery

I recorded the time `2026-10-02 10:43:47+03`, deleted all 17 rows (`DELETE 17`), forced a WAL switch, and confirmed the count was 0. I stopped PostgreSQL, moved the old data folder to `data_old`, and extracted the base backup into a new data folder.

I then added `restore_command`, `recovery_target_time = '2026-10-02 10:43:47+03'` and `recovery_target_action = 'promote'`, created `recovery.signal`, and started PostgreSQL. It replayed the archived WAL up to that time. The count returned to 17 and `pg_is_in_recovery` was false, so the point-in-time recovery worked.

![Step 3 time recorded and delete](images/step_5.png)
![Step 3 disaster confirmed](images/step_6.png)
![Step 3 data folder replaced](images/step_7.png)
![Step 3 recovery settings](images/step_8.png)
![Step 3 recovery verified](images/step_9.png)

## Step 4: Streaming standby

I created the `replicator` role with `REPLICATION LOGIN`, built the standby with `pg_basebackup -R` into `C:\standby`, removed the inherited recovery settings, set `port = 5433`, and started it as a service. `pg_is_in_recovery` returned true.

![Step 4 standby config](images/step_10.png)
![Step 4 standby running](images/step_11.png)

## Step 5: Replication health

`pg_stat_replication` showed `walreceiver` in the `streaming` state with a lag of 0 bytes. A row inserted on the primary appeared on the standby (both showed 18 rows).

![Step 5 replication status](images/step_12.png)
![Step 5 row counts match](images/step_13.png)

## Adaptations (Windows instead of Linux)

- `~/backups` became `%USERPROFILE%\backups`, and `cp` became `copy` with paths `C:\pgarchive` and `C:\pgbackup`, because PostgreSQL runs as a Windows service with no `~` or `sudo systemctl`.
- PostgreSQL 12+ needs a `recovery.signal` file to start recovery. I used `recovery_target_action = 'promote'` so the server opens for writes at the target.
- No `pg_hba.conf` edit was needed because the Windows installer already allows replication from 127.0.0.1.
- The standby runs on port 5433 because it shares the machine with the primary.
- My first standby promoted itself because it inherited the recovery settings from the primary. I rebuilt it without them and it streamed correctly.
