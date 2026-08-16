# Cadvisor容器部署

> 好像最新版的容器会出现部署之后无法正常运行的状态，也不知道为什么，本次选择的版本是lagoudocker/cadvisor:v0.37.0

* 直接一行命令：

  ~~~bash
  docker run \
    --volume=/:/rootfs:ro \
    --volume=/var/run:/var/run:ro \
    --volume=/sys:/sys:ro \
    --volume=/var/lib/docker/:/var/lib/docker:ro \
    --volume=/dev/disk/:/dev/disk:ro \
    --publish=8080:8080 \
    --detach=true \
    --name=cadvisor \
    --privileged \
    --device=/dev/kmsg \
    lagoudocker/cadvisor:v0.37.0
  ~~~

* 检查容器运行状态

  ~~~bash
  root@super-System-Product-Name:~# docker ps
  CONTAINER ID   IMAGE                          COMMAND                   CREATED          STATUS                             PORTS                                         NAMES
  5744f19de261   lagoudocker/cadvisor:v0.37.0   "/usr/bin/cadvisor -…"   10 seconds ago   Up 10 seconds (health: starting)   0.0.0.0:8080->8080/tcp, [::]:8080->8080/tcp   cadvisor
  ~~~

* 进入http://localhost:8080

  <img src="https://typora2icture.oss-cn-beijing.aliyuncs.com/img2/20250521140436.png" alt="image.png" style="zoom:33%;" />

  成功部署Cadvisor

# 在Prometheus中加入监听

* 在Prometheus的Prometheus.yml中加入

  ~~~yaml
  root@node4:~# vim /usr/local/prometheus/prometheus.yml
  # my global config
  global:
    scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
    evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
    # scrape_timeout is set to the global default (10s).
  
  # Alertmanager configuration
  alerting:
    alertmanagers:
    - static_configs:
      - targets:
        # - alertmanager:9093
  
  # Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
  rule_files:
    # - "first_rules.yml"
    # - "second_rules.yml"
  
  # A scrape configuration containing exactly one endpoint to scrape:
  # Here it's Prometheus itself.
  scrape_configs:
    # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
    - job_name: 'prometheus'
  
      # metrics_path defaults to '/metrics'
      # scheme defaults to 'http'.
  
      static_configs:
      - targets: ['localhost:9090']
  
    - job_name: 'edge1'
      static_configs:
        - targets: ['192.168.127.134:9100']
  
    - job_name: 'edge3'
      static_configs:
        - targets: ['192.168.127.137:9100']
  
    - job_name: 'edge4'
      static_configs:
        - targets: ['192.168.127.138:9100']
  
    - job_name: 'edge5'
      static_configs:
        - targets: ['192.168.127.139:9100']
  
    - job_name: 'cadvisor'
      static_configs:
        - targets: ['localhost:8080']
  # 新增
    - job_name: 'cadvisor110'
      static_configs:
        - targets: ['192.168.127.110:8080']
  
    - job_name: 'cadvisor111'
      static_configs:
        - targets: ['192.168.127.111:8080']
  
    - job_name: 'cadvisor112'
      static_configs:
        - targets: ['192.168.127.112:8080']
  ~~~

* 结果查看
<img src="https://typora2icture.oss-cn-beijing.aliyuncs.com/img2/3d64d0bc6120c1a84cddf1d0d6a642a.png" alt="3d64d0bc6120c1a84cddf1d0d6a642a.png" style="zoom:33%;" />




# 参考文章

* https://blog.csdn.net/qq_34556414/article/details/109754026



  