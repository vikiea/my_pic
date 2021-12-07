# SQL变动
```sql
ALTER TABLE `union2345`.`tab_user` 
ADD COLUMN `cooperation` tinyint(1) NOT NULL DEFAULT 0 COMMENT '合作方式 1网站 2软件' AFTER `custom_spread_url`,
ADD COLUMN `download_url` varchar(255) NOT NULL DEFAULT '' COMMENT '合作包下载链接' AFTER `cooperation`,
ADD COLUMN `up_status` tinyint(1) NOT NULL DEFAULT 0 COMMENT '用户报备状态 0 未报备 1 待审核 2 报备通过 3 报备不通过' AFTER `download_url`;


ALTER TABLE `union2345`.`tab_user_all` 
ADD COLUMN `cooperation` tinyint(1) NOT NULL DEFAULT 0 COMMENT '合作方式 1网站 2软件' AFTER `e2345_uid`,
ADD COLUMN `download_url` varchar(255) NOT NULL DEFAULT '' COMMENT '合作包下载链接' AFTER `cooperation`,
ADD COLUMN `up_status` tinyint(1) NOT NULL DEFAULT 0 COMMENT '用户报备状态 0 未报备 1 待审核 2 报备通过 3 报备不通过' AFTER `download_url`;
```