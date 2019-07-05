---
layout: post
title: "如何查看swap如何被程序启动"
description: ""
category: 系统管理
tags: []
---

```
cat << EOF > swapstat.sh
#!/bin/bash

echo -e "PID\t\tSwap\t\tProc_Name"

for pid in `ls -l /proc | grep ^d | awk '{ print $9 }'| grep -v [^0-9]`
do
        if [ $pid -eq 1 ];then continue;fi
        grep -q "Swap" /proc/$pid/smaps 2>/dev/null
        if [ $? -eq 0 ];then
                swap=$(grep Swap /proc/$pid/smaps \
                        | gawk '{ sum+=$2;} END{ print sum }')
                proc_name=$(ps aux | grep -w "$pid" | grep -v grep \
                        | awk '{ for(i=11;i<=NF;i++){ printf("%s ",$i); }}')
                if [ $swap -gt 0 ];then
                        echo -e "${pid}\t${swap}\t${proc_name}"
                fi
        fi
done | sort -k2 -n | awk -F'\t' '{
pid[NR]=$1;
size[NR]=$2;
name[NR]=$3;
}
END{
for(id=1;id<=length(pid);id++)
        {
                if(size[id]<1024)
                        printf("%-10s\t%15sKB\t%s\n",pid[id],size[id],name[id]);
                else if(size[id]<1048576)
                        printf("%-10s\t%15.2fMB\t%s\n",pid[id],size[id]/1024,name[id]);
                else
                        printf("%-10s\t%15.2fGB\t%s\n",pid[id],size[id]/1048576,name[id]);
                }
        }'

EOF
```

# 变更记录

|Why | Who | When |
|----|-----|------|
|创建文件,提醒日后完善内容|fishcired|2019.07.05|

