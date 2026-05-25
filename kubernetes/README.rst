Kubernetes clustering for Jasmin
################################

Go to Jasmin docs to get `detailed Kubernetes how-to <https://docs.jasminsms.com/en/latest/installation/index.html#install-k8s>`_.

Manifests(``simple-pods/``)
============================

- ``00-namespace.yml`` —— ``jasmin`` 命名空间
- ``jasmin.yml`` —— Jasmin 拆分部署:core / interceptord / dlrlookupd / dlrd / deliversmd + Service + ConfigMap
- ``rabbitmq.yml`` —— RabbitMQ broker(``rabbitmq-broker``)
- ``redis.yml`` —— Redis(``redis-server``)
- ``observability.yml`` —— 监控相关
- ``smppsimulator.yml`` —— SMPP 模拟器(压测用)
- ``jasmin-web-panel.yml`` —— **页面管理功能**:Web 面板(Django)+ Celery + PostgreSQL

Web Panel(页面管理功能)
=========================

``jasmin-web-panel.yml`` 把 jasmin-web-panel 这套 Web 管理界面接进集群,新增三个组件:

- ``jasmin-web``        —— Django/Gunicorn 管理界面(容器 ``:8000``)
- ``jasmin-web-celery`` —— Celery 后台任务 worker(broker = Redis)
- ``jasmin-webdb``      —— 面板自身的 PostgreSQL 业务库

它**复用**集群里已有的服务,无需重复部署:

- ``redis-server``  —— Django 缓存 + Celery broker(用 Redis DB 1,与 jasmin-core 的 DB 0 隔离)
- ``jasmin-cli:8990`` —— jCli telnet,面板通过它下发 用户/组/路由/连接器 等管理命令(主通道)
- ``jasmin-http-api`` / ``jasmin-smpp-api`` —— 可选的发送通道

部署(注意顺序)
----------------

依赖 ``jasmin.yml`` / ``redis.yml`` 已经部署。先建命名空间,再依赖项,最后面板::

    kubectl apply -f simple-pods/00-namespace.yml
    kubectl apply -f simple-pods/redis.yml
    kubectl apply -f simple-pods/rabbitmq.yml
    kubectl apply -f simple-pods/jasmin.yml
    kubectl apply -f simple-pods/jasmin-web-panel.yml

或一次性应用整个目录::

    kubectl apply -f simple-pods/

访问
----

面板通过 NodePort ``30800`` 暴露,浏览器打开 ``http://<任一节点IP>:30800``。
默认管理员账号由镜像 entrypoint 的 ``manage.py samples`` 在首次迁移后创建。

注意事项
--------

- **生产务必修改** ``jasmin-web-secret`` 里的 ``SECRET_KEY``,以及 ``jasmin-webdb-secret`` 的数据库口令。
- ``jasmin-webdb`` 默认用 ``hostPath``(``/opt/jasmin/webdb``)持久化,生产建议换成 StorageClass PVC。
- 提交日志(submit_log)页面需另跑 ``sms_logger`` 才有数据,默认为空;部署方法见
  ``jasmin-web-panel.yml`` 末尾的说明。
