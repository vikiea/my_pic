# SQL变动记录
```sql
ALTER TABLE `my_2345`.`pull_new_share_proportion` 
MODIFY COLUMN `laxin_type` tinyint(2) NOT NULL DEFAULT 0 COMMENT '拉新类型 0 未知类型， 1 老码手淘拉新， 2 老码手淘绑卡， 3 支付宝拉新， 4 支付宝绑卡， 5 京东app拉新， 6 玩赚星球拉新' AFTER `proportion_type`;
ALTER TABLE `my_2345`.`pull_new_share_proportion_record` 
MODIFY COLUMN `laxin_type` tinyint(2) NOT NULL DEFAULT 0 COMMENT '拉新类型 0 未知类型， 1 老码手淘拉新， 2 老码手淘绑卡， 3 支付宝拉新， 4 支付宝绑卡， 5 京东app拉新， 6 玩赚星球拉新' AFTER `proportion_type`;
```