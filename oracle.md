#### rman备份后利用duplicate做不完全恢复
优势
* 源数据库不停机
* 可以多次重复克隆直到找到需要的数据

条件   
1. 上次全备到恢复目标时间点的联机日志和归档日志完整   
2. 上次全备是有效的   
3. 克隆目标处于nomout状态   
步骤   
一般都是在本机新建一个实例，不完全恢复完成后，将其摧毁。

1. 准备`pfile`，建立需要的目录   
假定实例名为xxx。参数文件中的`undo_tablespace`要和源数据库一致。

    ```
    $ vi $ORACLE_HOME/db/initxxx.ora
        dbname=xxx
        db_recovery_file_dest=/u01/oracle/fast_recovery_area
        db_recovery_file_dest_size=4g
        undo_tablespace=undotbs1
        control_files=/u01/oracle/oradata/u1/control01.ctl
    $ mkdir $ORACLE_BASE/admin/xxx/adump
    $ mkdir $ORACLE_BASE/admin/xxx/dpump
    $ mkdir $ORACLE_BASE/oradata/xxx
    ```   
2. 确保能够登录到源库与目标库   
本地登录时，Oracle通过环境变量`$ORACLE_SID`作为登录的目标，所以同一个环境中只能登录进1个实例，远程登录无此限制。在我们的环境中，源数据库是已经配置好监听的，新建的目标数据库没有配置监听，为了减少操作的复杂性，登录源库时用远程登录，登录目标库时用本地登录。
    ```
    -- 修改环境变量
    $ export ORACLE_SID=xxx
    # 登录测试
    -- sqlplus / as sysdba
    # 应该能够连接到一个空闲实例。源数据库没有停机，所以空闲的实例就是xxx。
    ```
3. RMAN中进行duplicate   
启动克隆目标数据库到nomoun阶段。
```
$ sqlplus / as sysdba
SQL> startup nomount
```
用`RMAN`连接源数据库与目标数据库，连接成功后会看到源数据库updb，目标数据库状态为nomount。
```
$ rman target sys/oracle@updb auxiliary /
```
`RMAN`中执行不完全恢复。`set until time`用来设定恢复的时间点，可以多次尝试以找到正确的时间点。`duplicate target database`用来设置克隆参数，`db_file_name_convert`用来转换源库的datafile路径到目标库的路径，`logfile`设置目标库的日志文件，只需要2个就够了。
```
RMAN> run{
    set until time "to_date('2013-01-25 19:23:18','yyyy-mm-dd hh24:mi:ss')";
    duplicate target database to xxx
        db_file_name_convert=('/u01/oracle/oradata/updb/','/u01/oracle/oradata/xxx/')
        logfile '/u01/oracle/oradata/xxx/redo01.log' size 10m,'u01/oracle/oradata/xxx/redo02.log' size 10m;
    }
```
4. (optional)对源库进行日志归档   
`duplicate`过程中只使用归档日志，如果需要的日志尚未归档，则无法完成克隆操作，此时需要对源库进行归档操作。
```
alter system archive log current;
```
5. 验证数据库已复制到设置的时间点
登录到目标库，检查需要的数据是否存在，如果不存在的话，重新克隆到更晚的时间眯。
6. (optional)摧毁目标库   
恢复完成后，可以将目标库摧毁，只要删除对应的文件即可。
```
SQL> shutdown abort
$ rm -rf $ORACLE_BASE/admin/xxx
$ rm -rf $ORACLE_BASE/oradata/xxx
$ rm $ORACLE_HOME/db/*xxx*
```
