# SQL变动记录说明
- `pull_new_taobao_activity`增加`last_transfer_time`字段
- `pull_new_newuser_admin`增加`last_transfer_time`字段
- 增加新表
    - `pull_new_relation_transfer_record`, 渠道和KA转移记录表
    - `pull_new_ka_create_record`, KA创建记录表
    - `pull_new_ka_create_recor_success`, KA创建成功明细表
    - `pull_new_ka_create_record_error`, KA创建失败明细表

> 相关的字段说明在表结构comment里面均有说明
