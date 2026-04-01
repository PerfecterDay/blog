# ES数据导出
{docsify-updated}

### 数据导出备份
```
npm install elasticdump -g
elasticdump


elasticdump \
--input=http://elastic:xz3H7rrPCCxrbsdt@es-sg-vfp331pa70007h94i.public.elasticsearch.aliyuncs.com:9200/hk_jyb_market_code_table \
--output=hk_jyb_market_code_table_data.json \
--type=data
--limit=500 //每次导出的数据限制

elasticdump \
--input=http://elastic:xz3H7rrPCCxrbsdt@es-sg-vfp331pa70007h94i.public.elasticsearch.aliyuncs.com:9200/hk_jyb_market_code_table \
--output=hk_jyb_market_code_table_analyzer.json \
--type=analyzer


elasticdump \
--input=http://elastic:xz3H7rrPCCxrbsdt@es-sg-vfp331pa70007h94i.public.elasticsearch.aliyuncs.com:9200/hk_jyb_market_code_table \
--output=hk_jyb_market_code_table_mapping.json \
--type=mapping
```

### 数据迁移
```
elasticdump \
--input=http://elastic:xz3H7rrPCCxrbsdt@es-sg-vfp331pa70007h94i.public.elasticsearch.aliyuncs.com:9200/jyb_market_code_table \
--output=http://elastic:xz3H7rrPCCxrbsdt@es-sg-5r538yl4p0001cvbx.public.elasticsearch.aliyuncs.com:9200/jyb_market_code_table \
--limit=1000 \
--type=data
```



