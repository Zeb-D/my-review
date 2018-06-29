# Linux 定时任务 Crontab

https://blog.csdn.net/xiyuan1999/article/details/8160998

http://man.linuxde.net/ 

man 该指令偏手册，如man xx 会列出xx命令的使用手册



## 背景

`0 1 * * * /home/clean_es_index/clean_es_script.sh > /dev/null 2>&1`

/dev/null : 在类Unix系统中，/dev/null，或称空设备，是一个特殊的设备文件，它丢弃一切写入其中的数据（但报告写入操作成功），读取它则会立即得到一个EOF。

Linux的设备会以目录的形式挂载到主机上



[root@platform ~]# crontab -l
0 1 * * * /home/clean_es_index/clean_es_script.sh > /dev/null 2>&1

!/bin/bash

清理elasticsearch index，仅保存一个月数据

last_month=`date -d "last month" +"%Y.%m"`
last_month_curr_day=`date -d "last month" +"%Y.%m.%d"`
curr_month=`date +"%Y.%m"`
curr_month_last_day=`date -d "$(date -d "1 month" +"%Y%m01") -1 day" +"%Y.%m.%d"`
curr_day=`date +"%Y.%m.%d"`
log_dir="/home/clean_es_index"

打印日志

function print_log {
  echo -e "\n[$(date '+%Y-%m-%d %H:%M:%S')]" $@ >> ${log_dir}/${curr_month}.log
}

//清理上个月当天的Index

print_log "清理开始，正在执行curl -XDELETE" "localhost:9200/logstash-*-${last_month_curr_day}"
curl -XDELETE "localhost:9200/logstash-*-${last_month_curr_day}" >> ${log_dir}/${curr_month}.log

如果今天是本月最后一天，删除上个月所有的index

if [ ${curr_day} = ${curr_month_last_day} ]; then
  print_log "最后一天，正在执行curl -XDELETE" "localhost:9200/logstash-*-${last_month}.*"
  curl -XDELETE "localhost:9200/logstash-*-${last_month}.*" >> ${log_dir}/${curr_month}.log
fi
print_log "清理结束"

















