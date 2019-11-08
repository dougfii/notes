# logstash-input-jdbc同步数据库中的数据（全量 和 增量）

+ 安装ruby
```
# install ...
# gem sources --remove https://rubygems.org/
# gem sources -a http://gems.ruby-china.com/
# gem sources -l
# gem install bundler
# bundle config mirror.https://rubygems.org https://gems.ruby-china.com
```

+ 安装logstash-input-jdbc
```
# logstash-plugin install logstash-input-jdbc
```

+ 下载并拷贝数据库驱动到logstash配置目录下
```
# cp postgresql-42.2.8.jar logstash-7.4.0/config/postgresql-42.2.8.jar
```

+ 在logstash配置下编写es模板文件 customer-order.json
```
{
    "mappings": {
        "properties": {
            "id": {
                "type": "long"
            },
            "member_id": {
                "type": "long"
            },
            "finance_type": {
                "type": "long"
            },
            .
            .
            .
            "payment_status": {
                "type": "long"
            },
            "actual_order": {
                "type": "text"
            }
        }
    }
}
```

+ 在logstash配置目录下编写pipeline logstash.conf
```
input {
  jdbc {
    jdbc_driver_library => "E:\download\logstash-7.4.0\config\postgresql-42.2.8.jar"
    jdbc_driver_class => "org.postgresql.Driver"
    jdbc_connection_string => "jdbc:postgresql://localhost:5432/gomro_cloud"
    jdbc_user => "postgres"
    jdbc_password => "xxxxxx"
    schedule => "* * * * *"
    jdbc_paging_enabled => "true"
    jdbc_page_size => "50000"
    statement => "select * from t_customer_order where id > :sql_last_value and del=false order by id asc"
    record_last_run => true
    use_column_value => true
    tracking_column => id
    codec => plain { charset => "UTF-8"}
  }
}

output {
  elasticsearch {
    hosts => ["localhost:9200"]
    index => "customer-order"
    document_id => "%{id}"
    document_type => "_doc"
    template =>"E:\download\logstash-7.4.0\config\customer-order.json"
    template_name =>"customer-order"
    template_overwrite =>"true"
  }
  stdout {
	codec => rubydebug
  }
}
```

+ 执行
```
# bin/logstash -f config/logstash.conf
```