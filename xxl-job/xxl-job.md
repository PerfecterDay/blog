Admin 控制任务执行的 Controller
JobInfoController /trigger
JobApiController /registry

### Admin的 endpoint
curl -XPOST 'http://localhost:9000/xxl-job-admin/jobinfo/trigger' \
  -H 'Accept: application/json, text/javascript, */*; q=0.01' \
  -H 'Accept-Language: zh-CN,zh;q=0.9' \
  -H 'Cache-Control: no-cache' \
  -H 'Connection: keep-alive' \
  -H 'Content-Type: application/x-www-form-urlencoded; charset=UTF-8' \
  -H 'Cookie: XXL_JOB_LOGIN_IDENTITY=7b226964223a312c22757365726e616d65223a2261646d696e222c2270617373776f7264223a226531306164633339343962613539616262653536653035376632306638383365222c22726f6c65223a312c227065726d697373696f6e223a6e756c6c7d' \
  -H 'Origin: http://47.243.160.178:8000' \
  -H 'Pragma: no-cache' \
  -H 'Referer: http://47.243.160.178:8000/xxl-job-admin/jobinfo?jobGroup=2' \
  -H 'User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/114.0.0.0 Safari/537.36' \
  -H 'X-Requested-With: XMLHttpRequest' \
  --data-raw 'id=2&executorParam=&addressList=' \
  --compressed \
  --insecure


   http://127.0.0.1:9000/xxl-job-admin/api/registry



### Executor 的 endpoint
EmbedServer
EmbedHttpServerHandler

curl -XPOST http://localhost:9001/run \
-H 'XXL-JOB-ACCESS-TOKEN:default_token' \
--data-raw '{
	"jobId": 2,
	"executorHandler": "fundTradeDateSyncer",
	"executorParams": "",
	"executorBlockStrategy": "SERIAL_EXECUTION",
	"executorTimeout": 0,
	"logId": "9",
	"logDateTime": 1686185214070,
	"glueType": "BEAN",
	"glueSource": "",
	"glueUpdatetime": 1670467769000,
	"broadcastIndex": 0,
	"broadcastTotal": 1
}'

curl -XPOST http://10.176.112.27:9001/run \
-H 'XXL-JOB-ACCESS-TOKEN:default_token' \
--data-raw '{"jobId":2,"executorHandler":"readExcel","executorParams":"","executorBlockStrategy":"SERIAL_EXECUTION","executorTimeout":0,"logId":9,"logDateTime":1686185214070,"glueType":"BEAN","glueSource":"","glueUpdatetime":1670467769000,"broadcastIndex":0,"broadcastTotal":1}'