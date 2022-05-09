#基于 PITR 恢复

1.启用归档

```
	#启用归档模式并配置归档命令
	postgres=# ALTER SYSTEM SET archive_mode  = on;
	ALTER SYSTEM
	postgres=# ALTER SYSTEM SET archive_command  = 'cp %p /data2/archive/%f';
	ALTER SYSTEM
	postgres=# ALTER SYSTEM SET wal_level = 'replica';
	ALTER SYSTEM


	[postgres@server1 ~]$ pg_ctl  restart -D $PGDATA -l /tmp/logfile
	waiting for server to shut down.... done
	server stopped
	waiting for server to start.... done
	server started

	#切换归档，以保证归档成功
	postgres=# SELECT * FROM pg_switch_wal();
	 pg_switch_wal
	---------------
	 0/18D5580
	(1 row)
```

2.查看归档目录

```
	postgres=# \! ls /data2/archive
	000000010000000000000001

```

3.查看 pg_wal 目录

```
	postgres=# \! ls $PGDATA/pg_wal
	000000010000000000000001  000000010000000000000002  archive_status


```

4.创建表原始数据

```
	postgres=# CREATE TABLE tab_test(id integer,name varchar,gendate timestamp default current_timestamp);
	CREATE TABLE
	postgres=# INSERT INTO tab_test VALUES(1,'First');
	INSERT 0 1
	postgres=#  SELECT * FROM tab_test;
	 id | name  |          gendate
	----+-------+----------------------------
	  1 | First | 2022-05-09 15:52:56.258822

	(1 row)
```

5.做一次基础备份

```
	[postgres@server1 ~]$ pg_basebackup -Ft -Xs -Pv -U postgres -c fast -D /data2/backup/
	pg_basebackup: initiating base backup, waiting for checkpoint to complete
	pg_basebackup: checkpoint completed
	pg_basebackup: write-ahead log start point: 0/3000028 on timeline 1
	pg_basebackup: starting background WAL receiver
	pg_basebackup: created temporary replication slot "pg_basebackup_2314"
	25426/25426 kB (100%), 1/1 tablespace
	pg_basebackup: write-ahead log end point: 0/3000100
	pg_basebackup: waiting for background process to finish streaming ...
	pg_basebackup: syncing data to disk ...
	pg_basebackup: renaming backup_manifest.tmp to backup_manifest
	pg_basebackup: base backup completed

```

6.基础备份完成后再次为表插入数据

```
	postgres=# INSERT INTO tab_test VALUES(2,'After Backup');
	INSERT 0 1
	postgres=# SELECT * FROM tab_test;
	 id |     name     |          gendate
	----+--------------+----------------------------
	  1 | First        | 2022-05-09 15:52:56.258822
	  2 | After Backup | 2022-05-09 15:53:44.972818
	(2 rows)
```

7.手动切换日志

```
	postgres=# SELECT pg_switch_wal();
	 pg_switch_wal
	---------------
	 0/4000710
	(1 row)

```

8.查看归档和 pg_wal

```
	postgres=# \! ls /data2/archive/ -lrth
	total 65M
	-rw------- 1 postgres postgres 16M May  9 15:51 000000010000000000000001
	-rw------- 1 postgres postgres 16M May  9 15:53 000000010000000000000002
	-rw------- 1 postgres postgres 16M May  9 15:53 000000010000000000000003
	-rw------- 1 postgres postgres 338 May  9 15:53 000000010000000000000003.00000028.backup
	-rw------- 1 postgres postgres 16M May  9 15:54 000000010000000000000004
	postgres=# \! ls $PGDATA/pg_wal -lrth
	total 49M
	-rw------- 1 postgres postgres 16M May  9 15:53 000000010000000000000003
	-rw------- 1 postgres postgres 338 May  9 15:53 000000010000000000000003.00000028.backup
	-rw------- 1 postgres postgres 16M May  9 15:54 000000010000000000000004
	drwx------ 2 postgres postgres 133 May  9 15:54 archive_status
	-rw------- 1 postgres postgres 16M May  9 15:54 000000010000000000000005

```

9.再次插入一条数据

```
	--查询一次时间，以方便恢复第二条数据,此刻只有两条数据
		postgres=# SELECT * FROM current_timestamp;
		       current_timestamp
		-------------------------------
		 2022-05-09 15:55:13.727361+08
		(1 row)

		postgres=# SELECT * FROM tab_test;
		 id |     name     |          gendate
		----+--------------+----------------------------
		  1 | First        | 2022-05-09 15:52:56.258822
		  2 | After Backup | 2022-05-09 15:53:44.972818
		(2 rows)

	--然后再插入第三条数据
		postgres=# INSERT INTO tab_test VALUES(3,'Third');
		INSERT 0 1
		postgres=# SELECT * FROM tab_test;
		 id |     name     |          gendate
		----+--------------+----------------------------
		  1 | First        | 2022-05-09 15:52:56.258822
		  2 | After Backup | 2022-05-09 15:53:44.972818
		  3 | Third        | 2022-05-09 15:56:06.301407
		(3 rows)


		--这里手动执行一次完整的检查点
			postgres=# CHECKPOINT ;SELECT * FROM current_timestamp;
			CHECKPOINT
			       current_timestamp
			-------------------------------
			 2022-05-09 15:56:23.438648+08
			(1 row)

```

10.切换一次归档

```
		postgres=# SELECT * FROM pg_switch_wal();
		 pg_switch_wal
		---------------
		 0/5000210
		(1 row)


	--查询时间 （用来恢复包含第三条之前的数据）
		postgres=# SELECT * FROM current_timestamp;
		       current_timestamp
		-------------------------------
		 2022-05-09 15:56:48.871958+08
		(1 row)

```

11.查看归档和 pg_wal

```
		postgres=# \! ls -lrth /data2/archive/
		total 81M
		-rw------- 1 postgres postgres 16M May  9 15:51 000000010000000000000001
		-rw------- 1 postgres postgres 16M May  9 15:53 000000010000000000000002
		-rw------- 1 postgres postgres 16M May  9 15:53 000000010000000000000003
		-rw------- 1 postgres postgres 338 May  9 15:53 000000010000000000000003.00000028.backup
		-rw------- 1 postgres postgres 16M May  9 15:54 000000010000000000000004
		-rw------- 1 postgres postgres 16M May  9 15:56 000000010000000000000005
		postgres=#  \! ls -lrth $PGDATA/pg_wal
		total 49M
		-rw------- 1 postgres postgres 338 May  9 15:53 000000010000000000000003.00000028.backup
		-rw------- 1 postgres postgres 16M May  9 15:54 000000010000000000000007
		-rw------- 1 postgres postgres 16M May  9 15:56 000000010000000000000005
		drwx------ 2 postgres postgres  96 May  9 15:56 archive_status
		-rw------- 1 postgres postgres 16M May  9 15:56 000000010000000000000006


```

12.模拟删除第二条数据

```
	--删除第二条数据
		postgres=# DELETE FROM tab_test WHERE id = 2;
		DELETE 1
		postgres=# SELECT * FROM tab_test;
		 id | name  |          gendate
		----+-------+----------------------------
		  1 | First | 2022-05-09 15:52:56.258822
		  3 | Third | 2022-05-09 15:56:06.301407
		(2 rows)
```

13.模拟恢复

```
	--先查看归档
		postgres=# \! ls -lrth /data2/archive/
		total 81M
		-rw------- 1 postgres postgres 16M May  9 15:51 000000010000000000000001
		-rw------- 1 postgres postgres 16M May  9 15:53 000000010000000000000002
		-rw------- 1 postgres postgres 16M May  9 15:53 000000010000000000000003
		-rw------- 1 postgres postgres 338 May  9 15:53 000000010000000000000003.00000028.backup
		-rw------- 1 postgres postgres 16M May  9 15:54 000000010000000000000004
		-rw------- 1 postgres postgres 16M May  9 15:56 000000010000000000000005
		postgres=# \! ls -lrth $PGDATA/pg_wal
		total 49M
		-rw------- 1 postgres postgres 338 May  9 15:53 000000010000000000000003.00000028.backup
		-rw------- 1 postgres postgres 16M May  9 15:54 000000010000000000000007
		-rw------- 1 postgres postgres 16M May  9 15:56 000000010000000000000005
		drwx------ 2 postgres postgres  96 May  9 15:56 archive_status
		-rw------- 1 postgres postgres 16M May  9 15:57 000000010000000000000006

	--正常停止
		[postgres@server1 ~]$  pg_ctl  stop -D $PGDATA -l /tmp/logfile
		waiting for server to shut down.... done
		server stopped
		[postgres@server1 ~]$


	--再次查看WAL 归档和 pg_wal
		--正常停止自动做一次归档
			[postgres@server1 ~]$ ls -lrth /data2/archive/
			total 97M
			-rw------- 1 postgres postgres 16M May  9 15:51 000000010000000000000001
			-rw------- 1 postgres postgres 16M May  9 15:53 000000010000000000000002
			-rw------- 1 postgres postgres 16M May  9 15:53 000000010000000000000003
			-rw------- 1 postgres postgres 338 May  9 15:53 000000010000000000000003.00000028.backup
			-rw------- 1 postgres postgres 16M May  9 15:54 000000010000000000000004
			-rw------- 1 postgres postgres 16M May  9 15:56 000000010000000000000005
			-rw------- 1 postgres postgres 16M May  9 15:58 000000010000000000000006
			[postgres@server1 ~]$ ls -lrth $PGDATA/pg_wal
			total 49M
			-rw------- 1 postgres postgres 338 May  9 15:53 000000010000000000000003.00000028.backup
			-rw------- 1 postgres postgres 16M May  9 15:56 000000010000000000000008
			-rw------- 1 postgres postgres 16M May  9 15:58 000000010000000000000006
			-rw------- 1 postgres postgres 16M May  9 15:58 000000010000000000000007
			drwx------ 2 postgres postgres  96 May  9 15:58 archive_status

	--删除数据目录
		[postgres@server1 ~]$ cd $PGDATA
		[postgres@server1 pgdata]$ rm -rf *
```

14.检查进程是否都终止

```
	[postgres@server1 ~]$ ps -ef | grep postgres | grep -Ev "grep|ps|bash|postgres"
	#如果没有终止，杀掉进程
```

15.将删除的数据恢复

    恢复到 9 小节记录第一次查询的时间点，此刻包含第二条数据

16.拷贝基础备份到 $PGDATA

```
	[postgres@server1 pgdata]$ ls /data2/backup/
	backup_manifest  base.tar  pg_wal.tar
	[postgres@server1 pgdata]$ tar -xf /data2/backup/base.tar -C $PGDATA/
```

17.配置恢复命令和恢复时间点

```
	#这儿注意将之前的归档命令暂时注释掉，也可以不注释
		[postgres@server1 pgdata]$ tail -2 $PGDATA/postgresql.auto.conf
		restore_command = 'cp /data2/archive/%f %p'
		recovery_target_time = '2022-05-09 15:55:13.727361+08'

```

18.编辑恢复配置信号文件

```
	[postgres@server1 ~]$ touch $PGDATA/recovery.signal

```

19.启动数据库

```
	--为了观察明显，先清空日志
		[postgres@server1 pgdata]$ >/tmp/logfile
		[postgres@server1 pgdata]$ pg_ctl start -D $PGDATA -l /tmp/logfile
		waiting for server to start.... done
		server started
		[postgres@server1 pgdata]$ cat /tmp/logfile
		2022-05-09 16:01:29.736 CST [2359] LOG:  starting PostgreSQL 14.2 on x86_64-pc-linux-gnu, compiled by gcc (GCC) 8.5.0 20210514 (Red Hat 8.5.0-10), 64-bit
		2022-05-09 16:01:29.737 CST [2359] LOG:  listening on IPv6 address "::1", port 5532
		2022-05-09 16:01:29.737 CST [2359] LOG:  listening on IPv4 address "127.0.0.1", port 5532
		2022-05-09 16:01:29.737 CST [2359] LOG:  listening on Unix socket "/tmp/.s.PGSQL.5532"
		2022-05-09 16:01:29.739 CST [2360] LOG:  database system was interrupted; last known up at 2022-05-09 15:53:14 CST
		cp: cannot stat '/data2/archive/00000002.history': No such file or directory
		2022-05-09 16:01:29.745 CST [2360] LOG:  starting point-in-time recovery to 2022-05-09 15:55:13.727361+08
		2022-05-09 16:01:29.751 CST [2360] LOG:  restored log file "000000010000000000000003" from archive
		2022-05-09 16:01:29.764 CST [2360] LOG:  redo starts at 0/3000028
		2022-05-09 16:01:29.764 CST [2360] LOG:  consistent recovery state reached at 0/3000100
		2022-05-09 16:01:29.764 CST [2359] LOG:  database system is ready to accept read-only connections
		2022-05-09 16:01:29.771 CST [2360] LOG:  restored log file "000000010000000000000004" from archive
		2022-05-09 16:01:29.796 CST [2360] LOG:  restored log file "000000010000000000000005" from archive
		2022-05-09 16:01:29.809 CST [2360] LOG:  recovery stopping before commit of transaction 736, time 2022-05-09 15:56:06.301531+08
		2022-05-09 16:01:29.809 CST [2360] LOG:  pausing at the end of recovery
		2022-05-09 16:01:29.809 CST [2360] HINT:  Execute pg_wal_replay_resume() to promote.
		[postgres@server1 pgdata]$
```

20.验证数据

```
		postgres=# SELECT pg_wal_replay_resume();
		 pg_wal_replay_resume
		----------------------

		(1 row)

		postgres=# SELECT * FROM tab_test;
		 id |     name     |          gendate
		----+--------------+----------------------------
		  1 | First        | 2022-05-09 15:52:56.258822
		  2 | After Backup | 2022-05-09 15:53:44.972818
		(2 rows)
```

21.查看 TimeLine

```
	[postgres@server1 pgdata]$ pg_controldata   | grep -i timelineid
	Latest checkpoint's TimeLineID:       2
	Latest checkpoint's PrevTimeLineID:   1
```

22.确认归档目录和 pg_wal 目录下的 wal 日志

```
	[postgres@server1 ~]$ ls -lrth /data2/archive/
	total 97M
	-rw------- 1 postgres postgres 16M May  9 15:51 000000010000000000000001
	-rw------- 1 postgres postgres 16M May  9 15:53 000000010000000000000002
	-rw------- 1 postgres postgres 16M May  9 15:53 000000010000000000000003
	-rw------- 1 postgres postgres 338 May  9 15:53 000000010000000000000003.00000028.backup
	-rw------- 1 postgres postgres 16M May  9 15:54 000000010000000000000004
	-rw------- 1 postgres postgres 16M May  9 15:56 000000010000000000000005
	-rw------- 1 postgres postgres 16M May  9 15:58 000000010000000000000006
	-rw------- 1 postgres postgres  50 May  9 16:01 00000002.history
	[postgres@server1 ~]$ ls -lrth $PGDATA/pg_wal
	total 65M
	-rw------- 1 postgres postgres 16M May  9 16:01 000000020000000000000006
	-rw------- 1 postgres postgres 16M May  9 16:01 000000020000000000000007
	-rw------- 1 postgres postgres 16M May  9 16:01 000000010000000000000005
	-rw------- 1 postgres postgres  50 May  9 16:01 00000002.history
	drwx------ 2 postgres postgres  72 May  9 16:01 archive_status
	-rw------- 1 postgres postgres 16M May  9 16:02 000000020000000000000005
```

```
	--特别说明:
		pg_wal 目录中有一个 00000002.history 的文件，该文件为 TimeLine 标识文件，并且在完成第一次恢复后，自动会将新增的该标识文件归档到归档目录(前提是归档依旧生效),并且 pg_wal 目录下会以新的 timeline 标识运行数据库

		如下图表示


		<--------timeline 1------------------>
					 |
			 对应上面的三条数据
			 "First" "After Backup" "Third"

		执行了一次恢复后，                     <-----timeline 2------------->
													  包含 2 条数据
													  "First" "After Backup"


		--现在想要在 Timeline 2 的基础上，想要恢复 Timeline 1 中包含的三条数据的可行性探讨。
			如果数据库中出现一个新的数据库化身，那么以前的数据对于当前的化身则不可用，因此如果在该化身上执行恢复，
			仅仅是当前新的化身周期内操作的数据可以被恢复，化身以外的操作不可用。因此想要恢复三条完整数据，只能
			在 Timeline 1 上进行操作恢复

```

23.在 timeline 2 上进行操作

```
	#插入数据
		postgres=# INSERT INTO tab_test VALUES(4,'This item will be record in tli2');
		INSERT 0 1
		postgres=# SELECT * FROM pg_switch_wal();
		 pg_switch_wal
		---------------
		 0/5000528
		(1 row)

		postgres=# SELECT * FROM tab_test;
		 id |               name               |          gendate
		----+----------------------------------+----------------------------
		  1 | First                            | 2022-05-09 15:52:56.258822
		  2 | After Backup                     | 2022-05-09 15:53:44.972818
		  4 | This item will be record in tli2 | 2022-05-09 16:24:27.504229
		(3 rows)

	#假设此刻删除了 id = 1 的记录
		postgres=# SELECT * FROM current_timestamp;
		       current_timestamp
		-------------------------------
		 2022-05-09 16:25:34.959361+08

		(1 row)
		postgres=# DELETE FROM tab_test WHERE id = 1;
		DELETE 1
		postgres=# SELECT * FROM tab_test;
		 id |               name               |          gendate
		----+----------------------------------+----------------------------
		  2 | After Backup                     | 2022-05-09 15:53:44.972818
		  4 | This item will be record in tli2 | 2022-05-09 16:24:27.504229
		(2 rows)

	#停止数据库
		[postgres@server1 ~]$ pg_ctl  stop -D $PGDATA -l /tmp/logfile
		waiting for server to shut down.... done
		server stopped

	#执行恢复
		#取回 timeline 1 的基础备份

		#然后配置恢复文件

		#创建信号文件

		#启动数据库
			--启动过程没有报错，但是日志报错,报错如下:

			[postgres@server1 ~]$ >/tmp/logfile
			[postgres@server1 ~]$ pg_ctl  start -D $PGDATA -l /tmp/logfile
			waiting for server to start.... done
			server started
			[postgres@server1 ~]$ cat /tmp/logfile
			2022-05-09 16:50:09.847 CST [2629] LOG:  starting PostgreSQL 14.2 on x86_64-pc-linux-gnu, compiled by gcc (GCC) 8.5.0 20210514 (Red Hat 8.5.0-10), 64-bit
			2022-05-09 16:50:09.847 CST [2629] LOG:  listening on IPv6 address "::1", port 5532
			2022-05-09 16:50:09.847 CST [2629] LOG:  listening on IPv4 address "127.0.0.1", port 5532
			2022-05-09 16:50:09.848 CST [2629] LOG:  listening on Unix socket "/tmp/.s.PGSQL.5532"
			2022-05-09 16:50:09.849 CST [2630] LOG:  database system was interrupted while in recovery at log time 2022-05-09 15:53:14 CST
			2022-05-09 16:50:09.849 CST [2630] HINT:  If this has occurred more than once some data might be corrupted and you might need to choose an earlier recovery target.
			2022-05-09 16:50:09.856 CST [2630] LOG:  restored log file "00000002.history" from archive
			2022-05-09 16:50:09.856 CST [2630] LOG:  starting point-in-time recovery to 2022-05-09 16:47:02.768178+08
			2022-05-09 16:50:09.858 CST [2630] LOG:  restored log file "00000002.history" from archive
			2022-05-09 16:50:09.878 CST [2630] LOG:  restored log file "000000010000000000000003" from archive
			2022-05-09 16:50:09.888 CST [2630] LOG:  redo starts at 0/3000028
			2022-05-09 16:50:09.888 CST [2630] LOG:  consistent recovery state reached at 0/3000100
			2022-05-09 16:50:09.888 CST [2629] LOG:  database system is ready to accept read-only connections
			2022-05-09 16:50:09.895 CST [2630] LOG:  restored log file "000000010000000000000004" from archive
			2022-05-09 16:50:09.917 CST [2630] LOG:  restored log file "000000020000000000000005" from archive
			2022-05-09 16:50:09.936 CST [2630] LOG:  restored log file "000000020000000000000006" from archive
			2022-05-09 16:50:09.958 CST [2630] LOG:  restored log file "000000020000000000000007" from archive
			2022-05-09 16:50:09.980 CST [2630] LOG:  restored log file "000000020000000000000008" from archive
			cp: cannot stat '/data2/archive/000000020000000000000009': No such file or directory
			2022-05-09 16:50:09.997 CST [2630] LOG:  redo done at 0/80000F8 system usage: CPU: user: 0.00 s, system: 0.01 s, elapsed: 0.10 s
			2022-05-09 16:50:09.997 CST [2630] LOG:  last completed transaction was at log time 2022-05-09 16:38:29.545234+08
			2022-05-09 16:50:09.997 CST [2630] FATAL:  recovery ended before configured recovery target was reached
			2022-05-09 16:50:09.997 CST [2629] LOG:  startup process (PID 2630) exited with exit code 1
			2022-05-09 16:50:09.997 CST [2629] LOG:  terminating any other active server processes
			2022-05-09 16:50:09.998 CST [2629] LOG:  shutting down due to startup process failure
			2022-05-09 16:50:09.999 CST [2629] LOG:  database system is shut down

```

24.解决上面报错

```
	-----非常重要-----
		在第一次恢复完成以后，一定要重新备份数据库集簇
			[postgres@server1 ~]$ pg_basebackup -Ft -Xs -Pv -U postgres -c fast -D /data2/backup2/
			pg_basebackup: initiating base backup, waiting for checkpoint to complete
			pg_basebackup: checkpoint completed
			pg_basebackup: write-ahead log start point: 0/6000028 on timeline 2
			pg_basebackup: starting background WAL receiver
			pg_basebackup: created temporary replication slot "pg_basebackup_2501"
			25427/25427 kB (100%), 1/1 tablespace
			pg_basebackup: write-ahead log end point: 0/6000100
			pg_basebackup: waiting for background process to finish streaming ...
			pg_basebackup: syncing data to disk ...
			pg_basebackup: renaming backup_manifest.tmp to backup_manifest
			pg_basebackup: base backup completed

		--执行操作
			postgres=# INSERT INTO tab_test VALUES(4,'This item will be record in tli2');
			INSERT 0 1
			postgres=# SELECT * FROM pg_switch_wal();
			 pg_switch_wal
			---------------
			 0/7000218
			(1 row)

			postgres=# SELECT * FROM tab_test;
			 id |               name               |          gendate
			----+----------------------------------+----------------------------
			  1 | First                            | 2022-05-09 15:52:56.258822
			  2 | After Backup                     | 2022-05-09 15:53:44.972818
			  4 | This item will be record in tli2 | 2022-05-09 16:37:47.275977
			(3 rows)

		--然后删除数据（也就是误操作）

			postgres=# SELECT * FROM current_timestamp;
			       current_timestamp
			-------------------------------
			 2022-05-09 16:38:25.305327+08
			(1 row)

			postgres=# DELETE FROM tab_test WHERE id = 1;
			DELETE 1
			postgres=# SELECT * FROM tab_test;
			 id |               name               |          gendate
			----+----------------------------------+----------------------------
			  2 | After Backup                     | 2022-05-09 15:53:44.972818
			  4 | This item will be record in tli2 | 2022-05-09 16:37:47.275977
			(2 rows)

		--正常停止数据库
			[postgres@server1 ~]$ pg_ctl  stop -D $PGDATA -l /tmp/logfile
			waiting for server to shut down.... done
			server stopped

		--恢复过程
			--取回基础备份（基于 timeline2的基础备份）
				[postgres@server1 ~]$ rm -rf $PGDATA/*
				[postgres@server1 ~]$ tar -xf /data2/backup2/base.tar -C $PGDATA/

			--编辑配置文件
				[postgres@server1 ~]$ tail -3 $PGDATA/postgresql.auto.conf
				restore_command = 'cp /data2/archive/%f %p'
				recovery_target_time = '2022-05-09 16:38:25.305327+08'
				recovery_target_timeline = 2

			--创建信号文件
				[postgres@server1 ~]$ touch $PGDATA/recovery.signal

			--启动数据库
				[postgres@server1 ~]$ >/tmp/logfile
				[postgres@server1 ~]$ pg_ctl  start -D $PGDATA -l /tmp/logfile
				waiting for server to start.... done
				server started
				[postgres@server1 ~]$ cat /tmp/logfile
				2022-05-09 16:44:30.394 CST [2528] LOG:  starting PostgreSQL 14.2 on x86_64-pc-linux-gnu, compiled by gcc (GCC) 8.5.0 20210514 (Red Hat 8.5.0-10), 64-bit
				2022-05-09 16:44:30.394 CST [2528] LOG:  listening on IPv6 address "::1", port 5532
				2022-05-09 16:44:30.394 CST [2528] LOG:  listening on IPv4 address "127.0.0.1", port 5532
				2022-05-09 16:44:30.395 CST [2528] LOG:  listening on Unix socket "/tmp/.s.PGSQL.5532"
				2022-05-09 16:44:30.396 CST [2529] LOG:  database system was interrupted; last known up at 2022-05-09 16:37:17 CST
				2022-05-09 16:44:30.402 CST [2529] LOG:  restored log file "00000002.history" from archive
				2022-05-09 16:44:30.402 CST [2529] LOG:  starting point-in-time recovery to 2022-05-09 16:38:25.305327+08
				2022-05-09 16:44:30.404 CST [2529] LOG:  restored log file "00000002.history" from archive
				2022-05-09 16:44:30.411 CST [2529] LOG:  restored log file "000000020000000000000006" from archive
				2022-05-09 16:44:30.426 CST [2529] LOG:  redo starts at 0/6000028
				2022-05-09 16:44:30.426 CST [2529] LOG:  consistent recovery state reached at 0/6000100
				2022-05-09 16:44:30.426 CST [2528] LOG:  database system is ready to accept read-only connections
				2022-05-09 16:44:30.433 CST [2529] LOG:  restored log file "000000020000000000000007" from archive
				2022-05-09 16:44:30.451 CST [2529] LOG:  restored log file "000000020000000000000008" from archive
				2022-05-09 16:44:30.462 CST [2529] LOG:  recovery stopping before commit of transaction 738, time 2022-05-09 16:38:29.545234+08
				2022-05-09 16:44:30.462 CST [2529] LOG:  pausing at the end of recovery
				2022-05-09 16:44:30.462 CST [2529] HINT:  Execute pg_wal_replay_resume() to promote.


			--验证数据
				[postgres@server1 ~]$ psql
				psql (14.2)
				Type "help" for help.

				postgres=# SELECT pg_wal_replay_resume();
				 pg_wal_replay_resume
				----------------------

				(1 row)

				postgres=# select * from tab_test;
				 id |               name               |          gendate
				----+----------------------------------+----------------------------
				  1 | First                            | 2022-05-09 15:52:56.258822
				  2 | After Backup                     | 2022-05-09 15:53:44.972818
				  4 | This item will be record in tli2 | 2022-05-09 16:37:47.275977
				(3 rows)



```

25.id = 3 的数据如何恢复？

```

	由于 id = 3 的数据位于 timeline 1 上，因此对于它的恢复，就是拿 timeline 1 的备份进行恢复
	如下：

	[root@server1 ~]# tar -xf /data2/backup/base.tar -C /data1/pgdata1/
	[root@server1 ~]#

	--编辑配置文件
		[postgres@server1 ~]$ tail -4 /data1/pgdata1/postgresql.auto.conf
		restore_command = 'cp /data2/archive/%f %p'
		recovery_target_time = '2022-05-09 15:56:48.871958+08' #时间位置在 10 小节
		recovery_target_timeline = 1 #指定时间线
		port=5555

	--创建信号文件
		[postgres@server1 ~]$ touch /data1/pgdata1/recovery.signal

	--启动数据库
		[postgres@server1 ~]$ >/tmp/log5555
		[postgres@server1 ~]$ pg_ctl start -D /data1/pgdata1/ -l /tmp/log5555
		waiting for server to start.... done
		server started
		[postgres@server1 ~]$ cat /tmp/log5555
		2022-05-09 17:06:13.795 CST [2812] LOG:  starting PostgreSQL 14.2 on x86_64-pc-linux-gnu, compiled by gcc (GCC) 8.5.0 20210514 (Red Hat 8.5.0-10), 64-bit
		2022-05-09 17:06:13.795 CST [2812] LOG:  listening on IPv6 address "::1", port 5555
		2022-05-09 17:06:13.795 CST [2812] LOG:  listening on IPv4 address "127.0.0.1", port 5555
		2022-05-09 17:06:13.796 CST [2812] LOG:  listening on Unix socket "/tmp/.s.PGSQL.5555"
		2022-05-09 17:06:13.797 CST [2813] LOG:  database system was interrupted while in recovery at log time 2022-05-09 15:53:14 CST
		2022-05-09 17:06:13.797 CST [2813] HINT:  If this has occurred more than once some data might be corrupted and you might need to choose an earlier recovery target.
		2022-05-09 17:06:13.801 CST [2813] LOG:  starting point-in-time recovery to 2022-05-09 15:56:48.871958+08
		2022-05-09 17:06:13.807 CST [2813] LOG:  restored log file "000000010000000000000003" from archive
		2022-05-09 17:06:13.826 CST [2813] LOG:  redo starts at 0/3000028
		2022-05-09 17:06:13.826 CST [2813] LOG:  consistent recovery state reached at 0/3000100
		2022-05-09 17:06:13.827 CST [2812] LOG:  database system is ready to accept read-only connections
		2022-05-09 17:06:13.833 CST [2813] LOG:  restored log file "000000010000000000000004" from archive
		2022-05-09 17:06:13.850 CST [2813] LOG:  restored log file "000000010000000000000005" from archive
		2022-05-09 17:06:13.892 CST [2813] LOG:  restored log file "000000010000000000000006" from archive
		2022-05-09 17:06:13.906 CST [2813] LOG:  recovery stopping before commit of transaction 737, time 2022-05-09 15:57:38.329015+08
		2022-05-09 17:06:13.906 CST [2813] LOG:  pausing at the end of recovery
		2022-05-09 17:06:13.906 CST [2813] HINT:  Execute pg_wal_replay_resume() to promote.

	--验证数据
		[postgres@server1 ~]$ psql -p 5555
		psql (14.2)
		Type "help" for help.

		postgres=# select pg_wal_replay_resume() ;
		 pg_wal_replay_resume
		----------------------

		(1 row)

		postgres=# SELECT * FROM tab_test;
		 id |     name     |          gendate
		----+--------------+----------------------------
		  1 | First        | 2022-05-09 15:52:56.258822
		  2 | After Backup | 2022-05-09 15:53:44.972818
		  3 | Third        | 2022-05-09 15:56:06.301407
		(3 rows)

```
