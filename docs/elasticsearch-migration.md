# Elastic Search 数据迁移
1. elasticsearch-dump
2. snapshot
3. reindex
4. logstash

### elasticsearch-dump
+ 安装
```
# npm install elasticdump -g
```

+ 主要参数说明
```
 --input: 源地址，可为ES集群URL、文件或stdin,可指定索引，格式为：{protocol}://{host}:{port}/{index}
 --input-index: 源ES集群中的索引
 --output: 目标地址，可为ES集群地址URL、文件或stdout，可指定索引，格式为：{protocol}://{host}:{port}/{index}
 --output-index: 目标ES集群的索引
 --type: 迁移类型，默认为data,表明只迁移数据，可选settings, analyzer, data, mapping, alias
```

+ 迁移单个索引  
注意第一条命令先将索引的settings先迁移，如果直接迁移mapping或者data将失去原有集群中索引的配置信息如分片数量和副本数量等，当然也可以直接在目标集群中将索引创建完毕后再同步mapping与data
```
# elasticdump --input=http://10.0.8.1:9200/customer-order --output=http://10.0.3.77:9200/customer-order --type=settings
# elasticdump --input=http://10.0.8.1:9200/customer-order --output=http://10.0.3.77:9200/customer-order --type=mapping
# elasticdump --input=http://10.0.8.1:9200/customer-order --output=http://10.0.3.77:9200/customer-order --type=data
```

+ 迁移所有索引   
注意此操作并不能迁移索引的配置如分片数量和副本数量，必须对每个索引单独进行配置的迁移，或者直接在目标集群中将索引创建完毕后再迁移数据
```
# elasticdump --input=http://10.0.8.1:9200 --output=http://10.0.3.77:9200
```
### snapshot
snapshot api是Elasticsearch用于对数据进行备份和恢复的一组api接口，可以通过snapshot api进行跨集群的数据迁移，原理就是从源ES集群创建数据快照，然后在目标ES集群中进行恢复。需要注意ES的版本问题：
```
目标ES集群的主版本号(如7.4.0中的7为主版本号)要大于等于源ES集群的主版本号
5.x版本的集群创建的快照不能在7.x版本中恢复;
```

+ 源ES集群中创建repository   
创建快照前必须先创建repository仓库，一个repository仓库可以包含多份快照文件，repository主要有一下几种类型
```
fs: 共享文件系统，将快照文件存放于文件系统中
url: 指定文件系统的URL路径，支持协议：http,https,ftp,file,jar
s3: AWS S3对象存储,快照存放于S3中，以插件形式支持
hdfs: 快照存放于hdfs中，以插件形式支持
cos: 快照存放于腾讯云COS对象存储中，以插件形式支持
```
如果需要从自建ES集群迁移至腾讯云的ES集群，可以直接使用fs类型仓库，注意需要在Elasticsearch配置文件elasticsearch.yml设置仓库路径：
```
path.repo: ["/usr/local/services/test"]
```

+ 调用snapshot api创建repository
```
curl -XPUT http://10.0.8.1:9200/_snapshot/my_backup -H 'Content-Type: application/json' -d
'{
    "type": "fs",
    "settings": {
        "location": "/usr/local/services/test",
        "compress": true
    }
}'
```
如果需要从其它云厂商的ES集群迁移至腾讯云ES集群，或者腾讯云内部的ES集群迁移，可以使用对应云厂商他提供的仓库类型，如AWS的S3, 阿里云的OSS，腾讯云的COS等
```
curl -XPUT http://10.0.8.1:9200/_snapshot/my_s3_repository
{
    "type": "s3",
    "settings": {
        "bucket": "my_bucket_name",
        "region": "us-west"
    }
}
```

+ 源ES集群中创建snapshot  
调用snapshot api在创建好的仓库中创建快照
```
curl -XPUT http://10.0.8.1:9200/_snapshot/my_backup/snapshot_1?wait_for_completion=true
```
创建快照可以指定索引，也可以指定快照中包含哪些内容，具体的api接口参数可以查阅官方文档

+ 目标ES集群中创建repository  
目标ES集群中创建仓库和在源ES集群中创建仓库类似，用户可在腾讯云上创建COS对象bucket， 将仓库将在COS的某个bucket下

+ 移动源ES集群snapshot至目标ES集群的仓库  
把源ES集群创建好的snapshot上传至目标ES集群创建好的仓库中

+ 从快照恢复
```
curl -XPUT http://10.0.3.77:9200/_snapshot/my_backup/snapshot_1/_restore
```

+ 查看快照恢复状态
```
curl http://10.0.3.77:9200/_snapshot/_status
```

### reindex
reindex是Elasticsearch提供的一个api接口，可以把数据从源ES集群导入到当前的ES集群，同样实现了数据的迁移，限于腾讯云ES的实现方式，当前版本不支持reindex操作

+ 配置reindex.remote.whitelist参数  
需要在目标ES集群中配置该参数，指明能够reindex的远程集群的白名单

+ 调用reindex api  
以下操作表示从源ES集群中查询名为test1的索引，查询条件为title字段为elasticsearch，将结果写入当前集群的test2索引
```
POST _reindex
{
    "source": {
        "remote": {
            "host": "http://10.0.8.1:9200"
        },
        "index": "test1",
        "query": {
            "match": {
                "title": "elasticsearch"
            }
        }
    },
    "dest": {
        "index": "test2"
    }
}
```

### logstash
logstash支持从一个ES集群中读取数据然后写入到另一个ES集群，因此可以使用logstash进行数据迁移
```
input {
    elasticsearch {
        hosts => ["http://10.0.8.1:9200"]
        index => "*"
        docinfo => true
    }
}

output {
    elasticsearch {
        hosts => ["http://10.0.3.77:9200"]
        index => "%{[@metadata][_index]}"
    }
}
```
上述配置文件将源ES集群的所有索引同步到目标集群中，当然可以设置只同步指定的索引，logstash的更多功能可查阅logstash官方文档