#### 直接复制`datafile`+传输表空间
要求   
* 源数据库与目标数据库的字符集必须相同
* 两边的数据库名字要相同

迁移流程

1. 确认需要迁移的表空间可以为`read only`，只要可以`read only`就可以用这个方法迁移   
2. 用`exp`导出表空间的DDL语句

    ```
    SQL> alter tablespace jforum read only;
    $ exp \'/ as sysdba\' file=jforum.dmp tablespaces=jforum transport_tablespace=y
    ```
3. 如果目标平台与源平台不一致，需要用`rman`转换datafile的头部

    ```
    # 找出平台字符串，`rman`转换时要用到
    SQL> select platform_name, endian_format from v$transportable_platform;
    # 用rman转换datafile
    RMAN> convert tablespace jforum to platform 'PLATFORM_NAME' format '/u01/a    pp/oracle/jforum.dbf';
    ```
4. 拷贝导出的表空间file.dmp和datafile到目标机器上
5. 确认字符集与库名相同，如果不同需要重建控制文件
6. 建立源库中用到表空间的用户

    ```
    # 从源库中找出需要建立的用户
    SQL> select distinct owner from dba_tables where tablespace_name='JFORUM'
            union
        select distinct owner from dba_indexes where tablespace_name='JFORUM';
    # 建立用户
    SQL> grant connect,resource to USERNAME identified by PASSWORD;
    ```
7. 导入表空间，修改为新建用户的默认表空间

    ```
    # 导入时datafile要指定绝对路径
    $ imp \'/ as sysdba\' file=/uu01/oracle/oradata/jforum.dmp datafiles=/u01/oracle/oradata/jforum.dbf transport_tablespace=y
    SQL> alter user jforum default tablespace jforum;
    ```
