# SQL变动记录
```sql
ALTER TABLE `my_2345`.`pull_new_taobao_activity` 
ADD COLUMN `mt_adzone_id` varchar(30) CHARACTER SET latin1 COLLATE latin1_swedish_ci NOT NULL DEFAULT '' COMMENT '美团推广AdzoneId' AFTER `if_confirm`,
DROP PRIMARY KEY,
ADD PRIMARY KEY (`id`) USING BTREE;
```