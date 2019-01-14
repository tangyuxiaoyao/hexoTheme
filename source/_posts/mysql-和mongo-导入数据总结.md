---
title: mysql 和mongo 导入数据总结
date: 2018-04-04 15:17:22
tags: [mysql,mongo]
---
## 需求背景
>线上系统环境迁移，数据部分mysql和mongo需要迁移，大部分的数据通过dump，已经迁移过去，迁移过程中会有增量的数据产生，这部分数据要怎么做到无缝迁移。
<!--more-->
## 实现分析
>因为系统落地之前都会放到队列里面等待消费，然后落库，所以新系统保证服务正常以后，域名解析到新环境，然后流程环节知道队列，等待老版环境队列消费完了而且ngnix中无用户请求记录进来（因为域名解析会有几分钟的时间），我们开始提取增量的数据，mysql记录的是表记录的最大id，mongo记录也是最大的集合objectId，然后批量导出。

## 技术细节

### mysql

``` shell
#导出
mysql -uupsmart -p dbname -e 'select * from t where id>max(id)' >new_data.tsv
#导入
mysqlimport [options] db_name textfile1 ...

```
>tip①:max(id)为dump之前记录的最大id。
>tip②:mysqllimport导入的时候一定要指定-local，当然这部分属于options，或者写的是绝对路径否则会报错，默认找的是和mysqlimport命令同一路径下面的文件
>tip③:dbname紧贴要导入的文件，而且textfile1的前缀名即为表名（.之前的那货）


### mongo

``` shell
#导出
mongoexport -d dbanme -c collname --type=json -o new_data.csv --query='{"_id":"{"$gt":ObjectId("max(id)")}"}' 
#导入
mongoimport  -d dbanme -c collname  --type=json --file new_data.csv
```
>tip:本来打算是导出的时候指定csv格式然后导出，但是使用mongexport制定--type=csv的时候不能导出全部的字段，后来选择了默认的json格式导出。


