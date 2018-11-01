Celery&flower 后台运行部署
celery在后台运行
在生产环境中，你可能希望在后台运行worker.下面的文档中有详细的介绍：daemonization tutorial.
下面的脚本使用celery multi命令在后台启动一个或多个worker.
$ celery multi start w1 -A proj -l info 
celery multi v4.0.0 (0today8)
> Starting nodes...
> w1.halcyon.local: OK
> 你也可以重新启动:
> $ celery multi restart w1 -A proj -l info
> celery multi v4.0.0 (0today8)
> Stopping nodes...
> w1.halcyon.local: TERM -> 64024
> Waiting for 1 node.....
> w1.halcyon.local: OK
> Restarting node w1.halcyon.local: OK
> celery multi v4.0.0 (0today8)
> Stopping nodes...
> w1.halcyon.local: TERM -> 64052
> 或者停止它:
> $ celery multi stop w1 -A proj -l info
> stop命令是异步的，它不会等待所有的worker真正关闭。你可能需要使用stopwait命令，这个命令会保证当前所有的任务都执行完毕。
> $ celery multi stopwait w1 -A proj -l info
> 注:celery multi命令不会保存workers的信息，所以当重新启动时你需要使用相同的命令行参数。当停止时，只有相同的pidfile和logfile参数是必须的。

默认情况下，celery会在当前目录下创建pidfile和logfile.为了防止多个worker在启动时相互影响，你可以指定一个特定的目录。
$ mkdir -p /var/run/celery
$ mkdir -p /var/log/celery
$ celery multi start w1 -A proj -l info --pidfile=/var/run/celery/%n.pid \
--logfile=/var/log/celery/%n%I.log
通过multi命令你可以启动多个workers,另外这个命令还支持强大的命令行语法来为不同的workers指定不同的参数。
$ celery multi start 10 -A proj -l info -Q:1-3 images,video -Q:4,5 data \
-Q default -L:4,5 debug
更多关闭multi的举例请参考multi模块的API手册。