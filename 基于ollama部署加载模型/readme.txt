打开终端，进入 docker-compose.yml 文件所在目录，运行以下命令：

docker-compose down    # 停止并移除旧的容器
docker-compose up -d   # 根据新配置在后台启动服务

up -d 会自动拉取 open-webui 镜像并启动它

需要自己到容器内拉取模型：模型会直接下载到 docker-compose.yml 中指定的路径下
eg: docker exec -it deepseek-ollama ollama pull deepseek-r1:14b

清理流程：
完全卸载：docker compose down -v  --> 删除上文模型下载目录所有文件
更换删除模型：docker exec -it deepseek-ollama ollama rm deepseek-r1:14b

只作为本地模型使用，通过其他如vs中continue等类似接入ai模型的模块或软件，
注释掉open-webui:即可