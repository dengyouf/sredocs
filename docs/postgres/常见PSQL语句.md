- 查看psql版本

```shell
 ~]# su - postgres
-bash-4.2$ psql -c "SELECT version();"
                                                 version                                                  
----------------------------------------------------------------------------------------------------------
 PostgreSQL 14.17 on x86_64-pc-linux-gnu, compiled by gcc (GCC) 4.8.5 20150623 (Red Hat 4.8.5-44), 64-bit
```