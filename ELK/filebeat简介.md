# Filebeat 简介

https://www.elastic.co/guide/en/beats/filebeat/current/running-on-docker.html


1. `docker` pull docker.elastic.co/beats/filebeat:8.4.1`
2. 执行 setup: ```docker run \
	--net elastic \
	docker.elastic.co/beats/filebeat:8.4.1 \
	setup -E setup.kibana.host=kibana:5601 \
	-E output.elasticsearch.hosts="https://es01:9200"```
3. 通过卷配置filebeat，示例配置文件：`curl -L -O https://raw.githubusercontent.com/elastic/beats/8.4/deploy/docker/filebeat.docker.yml`
4. 
   ```docker run -d \
  --name=filebeat \
  --net elastic \
  --user=root \
  --volume="$(pwd)/filebeat.docker.yml:/usr/share/filebeat/filebeat.yml:ro" \
  --volume="/var/lib/docker/containers:/var/lib/docker/containers:ro" \
  --volume="/var/run/docker.sock:/var/run/docker.sock:ro" \
  --volume="/Users/coder_wang/Workspace/demo/log:/var/log" \
  docker.elastic.co/beats/filebeat:8.4.1 filebeat -e --strict.perms=false \
  -E output.elasticsearch.hosts=https://es01:9200```


ES 连接配置：
```
# filebeat.config:
#   modules:
#     path: ${path.config}/modules.d/*.yml
#     reload.enabled: false
filebeat.inputs:
- type: log
  paths:
    - /var/log/*.log

filebeat.autodiscover:
  enabled: false
  # providers:
  #   - type: docker
  #     hints.enabled: true

processors:
- add_cloud_metadata: ~


output.elasticsearch:  # 输出到elasticsearch
  hosts: ["https://10.187.144.42:8200"]
  username: "elastic"
  password: "izG+j-CKA8jLfi3Ds05I"
  ssl:
    enabled: true
    verification_mode: none

# output.console:
#  enabled: true
#  pretty: true


# output.elasticsearch:
#  hosts: ["https://myEShost:9200"]
#  username: "filebeat_internal"
#  password: "YOUR_PASSWORD" 
#  indices:
#    - index: "skills_consumer_%{+yyyy.MM.dd}"
#      when.equals:
#        fields.type: "skills_consumer"
#    - index: "skills_storage_%{+yyyy.MM.dd}"
#      when.equals:
#        fields.type: "skills_storage"
#  ssl:
#    enabled: true
#    ca_trusted_fingerprint: "b9a10bbe64ee9826abeda6546fc988c8bf798b41957c33d05db736716513dc9c"
    # verification_mode: none 
```
如果不知道 `ca_trusted_fingerprint` 然后报证书不信任错误：verification_mode: none 


测试配置文件是否正确：`./filebeat test config -e`

启动 filebeat 
```
sudo chown root filebeat.yml 
sudo chown root modules.d/nginx.yml 
sudo ./filebeat -e
# sudo ./filebeat -e -v -d '*'    debug模式启动
```