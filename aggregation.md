### Aggregation

```json
//简单的聚合
GET /kibana_sample_data_flights/_search
{
	"size":0,
	"aggs": {
		"flight_dest": {
			"terms": {
				"field": "DestCountry"
			}
		}
	}
}
//简单聚合后统计数值
GET /kibana_sample_data_flights/_search
{
	"size":0,
	"aggs": {
		"flight_dest": {
			"terms": {
				"field": "DestCountry"
			},
			"aggs": {
				"average_price": {
					"avg": {
						"field": "AvgTicketPrice"
					}
				},
				"max_price": {
					"max": {
						"field": "AvgTicketPrice"
					}
				},
				"min_price": {
					"min": {
						"field": "AvgTicketPrice"
					}
				}
			}
		}
	}
}

//支持嵌套
GET /kibana_sample_data_flights/_search
{
	"size":0,
	"aggs": {
		"flight_dest": {
			"terms": {
				"field": "DestCountry"
			},
			"aggs": {
				"price_stats": {
					"stats": {
						"field": "AvgTicketPrice"
					}
				},
				"weather": {
					"terms": {
						"field": "DestWeather",
						"size":3
					}
				}
			}
		}
	}
}
```

#### bucket & metrics

```json
POST employees/_search
{
	"size":0,
	"aggs": {
		"min_salary": {
			"min": {
				"field":"salary"
			}
		},
		"max_salary": {
			"max": {
				"field":"salary"
			}
		}
	}
}
POST employees/_search
{
	"size":0,
	"aggs": {
		"stats_salary": {
			"stats": {
				"field": "salary"
			}
		}
	}
}
//对keyword进行聚合
POST employees/_search
{
	"size":0,
	"aggs": {
		"jobs": {
			"term": {
				"field":"job.keyword"
			}
		}
	}
}
//对Text字段进行Term聚合会报错
//打开fielddata后查询，结果为对job字段进行分词后的结果
POST employees/_search
{
	"size":0,
	"aggs": {
		"jobs": {
			"term": {
				"field":"job"
			}
		}
	}
}
//需要对字段打开fielddata才可以
PUT employees/_mapping
{
	"properties": {
		"job": {
			"type": "text",
			"fielddata": true
		}
	}
}
//返回总共有多少个类别，分别对keyword和text结果是不一样的，因为对text有分词
POST employees/_search
{
	"size":0,
	"aggs": {
		"cardinate": {
			"cardinality": {
				"field": "job.keyword"
				//"field": "job"
			}
		}
	}
}

//不同工种中，年龄最大的3个员工的具体信息
POST employees/_search
{
	"size": 0,
	"aggs": {
		"jobs": {
			"terms": {
				"field": "job.keyword"
			}
		},
		"aggs": {
			"old_employee": {
				"top_hits": {
					"size": 3,
					"sort": [
						{"age":{"order":"desc"}}
					]
				}
			}
		}
	}
}

//优化Term聚合的性能，能够提前进行计算
PUT index
{
	"mappings": {
		"properties": {
			"foo": {
				"type":"keyword",
				"eager_global_ordinals": true
			}
		}
	}
}

//Ranges分桶，key是子桶的名字
POST employees/_search
{
	"size": 0,
	"aggs": {
		"salary_range": {
			"range": {
				"field": "salary",
				"ranges": [
					{"to": 10000},
					{"from": 10000, "to": 20000},
					{"from": 20000, "key": ">20000"}
				]
			}
		}
	}
}

//histrogram, 从0到10万，以5000一个区间进行分桶
POST employees/_search
{
	"size": 0,
	"aggs": {
		"salary_histrogram": {
			"histrogram": {
				"field": "salary",
				"interval": 5000,
				"extended_bounds": {
					"min":0,
					"max":100000
				}
			}
		}
	}
}

//嵌套聚合1， 按照工作类型分桶，并统计工资信息
POST employees/_search
{
	"size": 0,
	"aggs": {
		"job_salary_stats": {
			"term": {
				"field": "job.keyword"
			},
			"aggs": {
				"salary": {
					"stats": {
						"field": "salary"
					}
				}
			}
		}
	}
}

//嵌套2， 根据工作类型分桶，然后按照性别分桶，计算工资的统计信息
POST employees/_search
{
	"size": 0,
	"aggs": {
		"job_gender_stats": {
			"term": {
				"field": "job.keyword"
			},
			"aggs": {
				"gender_stats": {
					"term": {
						"field": "gender"
					},
					"aggs": {
						"salary_stats": {
							"stats": {
								"field": "salary"
							}
						}
					}
				}
			}
		}
	}
}
```



#### pipeline

对聚合分析的结果再进行聚合分析，pipeline的分析结果根据位置的不同分为两类：sibling（结果和现有结果同级），parent（结果内嵌到现有分析结果里面）

##### Sibling

```json
//平均工资最低的工作类型，还有max_bucket，avg_bucket，stats_bucket
POST employees/_search
{
	"size": 0,
	"aggs": {
		"jobs": {
			"terms": {
				"filed": "job.keyword"
			},
			"aggs": {
				"avg_salary": {
					"avg": {
						"field": "salary"
					}
				}
			}
		},
		"min_salary_by_job": {
			"min_bucket": {
				"buckets_path": "jobs>avg_salary"
			}
		}

	}
}
//平均工资的百分位数
POST employees/_search
{
	"size": 0,
	"aggs": {
		"jobs": {
			"terms": {
				"filed": "job.keyword"
			},
			"aggs": {
				"avg_salary": {
					"avg": {
						"field": "salary"
					}
				}
			}
		},
		"percentitles_salary_by_job": {
			"percentiles_bucket": {
				"buckets_path": "jobs>avg_salary"
			}
		}

	}
}
```

##### Parent

```json
//按照年龄对平均工资求导
//cumulative_sum(累计求和)
POST employees/_search
{
	"size": 0,
	"aggs": {
		"age_his": {
			"histogram": {
				"field": "age",
				"interval": 1,
				"min_doc_count": 1
			},
			"aggs": {
				"avg_salary": {
					"avg": {
						"field": "salary"
					}
				},
				"derivative_avg_salary": {
					"derivative": {
						"buckets_path": "avg_salary"
					}
				}
			}
		}
	}
}

//moving function（移动平均）
POST employees/_search
{
	"size": 0,
	"aggs": {
		"age": {
			"histogram": {
				"field": "age",
				"interval": 1,
				"min_doc_count": 1
			},
			"aggs": {
				"avg_salary": {
					"avg": {
						"field": "salary"
					}
				},
				"moving_avg_salary": {
					"moving_fn": {
						"buckets_path": "avg_salary",
						"window": 10,
						"script": "MovingFunctions.min(values)"
					}
				}
			}
		}
	}
}
```

### 聚合的作用范围

Elasticsearch聚合的默认作用范围是query查询的结果集，还支持filter，global，post filter改变聚合的作用范围

```json
//默认聚合的作用范围是query的结果
POST employees/_search
{
	"size": 0,
	"query": {
		"range": {
			"age": {"gte":40}
		}
	},
	"aggs": {
		"jobs": {
			"terms": {
				"field": "job.keyword"
			}
		}
	}
}

//Filter
POST employees/_search
{
	"size": 0,
	"aggs": {
		"all_jobs": {
			"terms": {
				"field": "job.keyword"
			}
		},
		"old_person": {
			"filter": {
				"range": {
					"age": {"from":35}
				}
			},
			"aggs": {
				"jobs": {
					"terms": {
						"field": "job.keyword"
					}
				}
			}
		}
	}
}

//Post filter，找到所有的job类型，还能找到聚合后符合条件的结果
POST employees/_search
{
	"size": 0,
	"aggs": {
		"jobs": {
			"terms": {
				"field": "job.keyword"
			}
		}
	},
	"post_filter": {
		"match": {
			"job.keyword": "Dev Manager"
		}
	}
}

//global，不受query的查询结果限制，作用的全部的文档
POST employees/_search
{
	"size": 0,
	"query": {
		"range": {
			"age": {"gte": 40}
		}
	},
	"aggs": {
		"jobs": {
			"terms": {
				"field": "job.keyword"
			}
		},
		"all": {
			"global": {},
			"aggs": {
				"avg_salary": {
					"avg": {
						"field": "salary"
					}
				}
			}
		}
	}
}
```



### 聚合中的排序

```json
POST employees/_search
{
	"size": 0,
	"query": {
		"range": {
			"age": {"gte": 20}
		}
	},
	"aggs": {
		"jobs": {
			"terms": {
				"field": "job.keyword",
				"order": [
					{"_count": "asc"},
					{"_key": "desc"}
				]
			}
		}
	}
}

//可以根据计算的聚合结果进行排序
POST employees/_search
{
	"size": 0,
	"aggs": {
		"jobs": {
			"terms": {
				"field": "job.keyword",
				"order": [
					{"avg_salary": "desc"}
				]
			},
			"aggs": {
				"avg_salary": {
					"avg": {
						"field": "salary"
					}
				}
			}
		}
	}
}

//可以根据计算的聚合结果(stats)进行排序
POST employees/_search
{
	"size": 0,
	"aggs": {
		"jobs": {
			"terms": {
				"field": "job.keyword",
				"order": [
					{"stats_salary.min": "desc"}
				]
			},
			"aggs": {
				"stats_salary": {
					"stats": {
						"field": "salary"
					}
				}
			}
		}
	}
}
```



### 聚合的结果不准确

Coordinating Node从其他分片获得的聚合数据有可能有缺漏，因为其他分片没有全局的信息

Terms Aggregation的返回结果中有两个值`doc_count_error_upper_bound`和`sum_other_doc_count`
- doc_count_error_upper_bound：被遗漏的term分桶，包含的文档，有可能的最大值
- sum_other_doc_count：除了返回结果bucket的terms以外，其他term的文档总数（总数-返回的总数）

#### 解决结果不准确的问题

- 当数据量不大的时候，设置primary shard为1
- 设置`shard_size`参数，提高精确度，其作用是每次从shard上额外多获取数据，打开`"term":{show_term_doc_count_error: true}`查看结果的准确信息

