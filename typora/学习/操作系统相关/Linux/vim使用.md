### 查找替换

1. `:s/vivian/sky/` 替换当前行第一个 vivian 为 sky 
   `:s/vivian/sky/g `替换当前行所有 vivian 为 sky 　 

2. `:n,$s/vivian/sky/` 替换第 n 行开始到最后一行中每一行的第一个 vivian 为 sky `:n,$s/vivian/sky/g `替换第 n 行开始到最后一行中每一行所有 vivian 为 sky 

       （n 为数字，若 n 为 .，表示从当前行开始到最后一行） 　 

3. `:%s/vivian/sky/`（等同于 `:g/vivian/s//sky/`） 替换每一行的第一个 vivian 为 sky `:%s/vivian/sky/g`（等同于 `:g/vivian/s//sky/g`） 替换每一行中所有 vivian 为 sky 　 

4. 可以使用 # 作为分隔符，此时中间出现的 / 不会作为分隔符

     `:s#vivian/#sky/# `替换当前行第一个 vivian/ 为 sky/ 
     　 
