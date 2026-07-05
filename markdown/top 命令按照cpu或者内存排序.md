# top 命令按照cpu或者内存排序

```
top -b -o +RES |head -n 20 | sed -n '1,$p'

top -b -o +%CPU |head -n 20|sed -n '8,$p'|grep -v ksmd|awk '{print $1}'
```