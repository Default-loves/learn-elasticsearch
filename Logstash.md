### 总述

- ELT工具，多数据源输入处理，输出到多数据源
- Pipeline：包含了input-filter-output三个阶段的处理流程
- Logstash Event：数据在内部流转时的具体表现形式，数据在input阶段被转换为Event，在output被转化成目标格式数据；Event其实是一个Java Object
- Codec：将源数据decode为Event，将Event encode为目标数据
- Queue：将源数据decode为Event后会放入Queue，给Filter进行消费。有Memory Queue（进程crash，机器宕机，都会引起数据丢失）和Persistent Queue（数据保存在磁盘中，保证数据会被消费）

### 例子

#### 多Pipeline实例

```json
- pipeline.id: my-pipeline-1
  path.config: "/etc/path/p1.config"
  pipeline.workers: 3
- pipeline.id: my-pipeline-2
  path.config: "/etc/path/p2.config"
  queue.type: persisted
```

#### Single Line例子
```json
//对输入包装一下输出
bin/logstash -e "input{stdin{codec=>line}}output{stdout{codec=> rubydebug}}"
//输入是JSON形式的数据
bin/logstash -e "input{stdin{codec=>json}}output{stdout{codec=> rubydebug}}"
//输出是一个"."，一般用来查看程序的进度
bin/logstash -e "input{stdin{codec=>line}}output{stdout{codec=> dots}}"
```

#### Multiline
```json
input {
	stdin {
		codec=> multiline {
			pattern=> "^/s"
			what=> "previous"
		}
	}
}
//输入一个错误日志
Exception in thread "main" java.lang.NullPointerException
	at com.example.myproject.Book.getTitle(book.java:16)
	at com.example.myproject.Book.getTitle(book.java:16)
Hello world
```

#### 一个具体的例子

```json
input {
  file {
    path => "/Users/yiruan/dev/elk7/logstash-7.0.1/bin/movies.csv"
    start_position => "beginning"
    sincedb_path => "/dev/null"
  }
}
filter {
  csv {
    separator => ","
    columns => ["id","content","genre"]
  }

  mutate {
    split => { "genre" => "|" }
    remove_field => ["path", "host","@timestamp","message"]
  }

  mutate {
    split => ["content", "("]
    add_field => { "title" => "%{[content][0]}"}
    add_field => { "year" => "%{[content][1]}"}
  }

  mutate {
    convert => {
      "year" => "integer"
    }
    strip => ["title"]
    remove_field => ["path", "host","@timestamp","message","content"]
  }

}
output {
   elasticsearch {
     hosts => "http://localhost:9200"
     index => "movies"
     document_id => "%{id}"
   }
  stdout {}
}
```

