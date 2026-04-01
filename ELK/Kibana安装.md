# Kibana


1. docker pull docker.elastic.co/kibana/kibana:8.4.1
2. sudo docker run -d --name kb01 --net elastic -p 8601:8601 -p 8701:5601 -e elasticsearch.hosts='https://es01:9200' kibana:8.4.2


kibana 设置 enrollment：
bin/kibana-setup --enrollment-token <enrollment-token>
