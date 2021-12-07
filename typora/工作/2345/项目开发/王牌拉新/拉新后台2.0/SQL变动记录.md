# SQL变动记录
- ```sql
ALTER TABLE `my_2345`.`pull_new_taobao_activity` 
ADD COLUMN `last_transfer_time` datetime NOT NULL DEFAULT '0000-00-00 00:00:00' COMMENT '最近一次转移时间' AFTER `mt_adzone_id`;
```

- 增加新表'pull_new_relation_transfer_record'
- ```sql
    ALTER TABLE `my_2345`.`pull_new_newuser_admin` 
ADD COLUMN `last_trancfer_time` datetime NOT NULL DEFAULT '0000-00-00 00:00:00' COMMENT '最近一次转移时间' AFTER `remark`;
```
- 增加新表 `pull_new_ka_create_record`,`pull_new_ka_create_recor_success`,`pull_new_ka_create_record_error`

