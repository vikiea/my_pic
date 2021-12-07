# Linux常用操作



### Linux下打印第N行

#### 实现

```shell
#第一种
cat filename | head -n 5 | tail -n +5

#第二种
sed -n '5p' filename
```

#### 扩展:打印第3~5行

```shell
#第一种
cat filename | head -n 5 | tail -n +3

#第二种
sed -n '3p;4p;5p' filename

#第三种
sed -n '3,5p' filename 
```

