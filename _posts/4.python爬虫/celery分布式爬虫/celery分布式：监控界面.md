celery分布式：监控界面

flower监控服务器
flower安装与运行的方法

通过浏览器打开地址

可能遇到的错误
pymongo.errors.NotMasterError: not master
关闭mongodb的数据集
其他
当服务器数量较多的时候，管理起来会很不方便，可以使用python的supervisor来管理后台进程，遗憾的是它并不支持python3，不过也可以装在python2的环境
虽然用了supervisor可以很方便的管理python程序，但是还是得一个个登陆不同的服务器的去管理，咋办捏？
我在github上找到一个工具supervisor-easy，可以批量管理supervisor，如图:

地址：https://github.com/trytofix/supervisor-easy