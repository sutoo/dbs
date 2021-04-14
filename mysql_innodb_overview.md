参考：
https://dev.mysql.com/doc/refman/5.7/en/innodb-architecture.html
https://michael.bouvy.net/post/understanding-mysql-innodb-buffer-pool-size

# in memory

## Buffer Pool
InnoDB data is stored in 16 KB pages (blocks), either on disk (ibdata files) or in memory (buffer pool).
innodb_buffer_page表可以用来查看buffer pool的使用情况。

Page types in buffer pool
```
select
page_type as Page_Type,
sum(data_size)/1024/1024 as Size_in_MB
from information_schema.innodb_buffer_page
group by page_type
order by Size_in_MB desc;
```

Buffer pool usage per index
```
select
table_name as Table_Name, index_name as Index_Name,
count(*) as Page_Count, sum(data_size)/1024/1024 as Size_in_MB
from information_schema.innodb_buffer_page
group by table_name, index_name
order by Size_in_MB desc;
```

14.5.2 Change Buffer
14.5.3 Adaptive Hash Index
14.5.4 Log Buffer
  
