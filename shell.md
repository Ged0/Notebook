# grep
grep -n "string" filename
 -A 匹配位置后多输出几行
 -B 匹配位置前多输出几行
 -C  ==  -A -B

# awk

```

 l | awk 'BEGIN {print "start"} {print $9; print NR}  END {print "end"}'

```

先执行BEGIN -> 读取文件 -> 对每行都执行主体 -> 执行END

## 内置变量:
- ARGC               命令行参数个数
- ARGV               命令行参数排列
- ENVIRON            支持队列中系统环境变量的使用
- FILENAME           awk浏览的文件名
- FNR                浏览文件的记录数
- FS                 设置输入域分隔符，等价于命令行 -F选项
- NF                 浏览记录的域的个数
- NR                 已读的记录数
- OFS                输出域分隔符
- ORS                输出记录分隔符
- RS                 控制记录分隔符

 
