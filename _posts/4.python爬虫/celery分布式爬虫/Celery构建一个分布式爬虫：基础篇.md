Celery构建一个分布式爬虫：基础篇

如何构建一个分布式爬虫：基础篇
首先，我们新建目录distributedspider，然后再在其中新建文件workers.py,里面内容如下
from celery import Celery
app = Celery('crawl_task', include=['tasks'], broker='redis://223.129.0.190:6379/1', backend='redis://223.129.0.190:6379/2')
# 官方推荐使用json作为消息序列化方式
app.conf.update(
CELERY_TIMEZONE='Asia/Shanghai',
CELERY_ENABLE_UTC=True,
CELERY_ACCEPT_CONTENT=['json'],
CELERY_TASK_SERIALIZER='json',
CELERY_RESULT_SERIALIZER='json',
)
上述代码主要是做Celery实例的初始化工作，include是在初始化celery app的时候需要引入的内容，主要就是注册为网络调用的函数所在的文件。然后我们再编写任务函数，新建文件tasks.py,内容如下
import requests
from bs4 import BeautifulSoup
from workers import app
@app.task
def crawl(url):
print('正在抓取链接{}'.format(url))
resp_text = requests.get(url).text
soup = BeautifulSoup(resp_text, 'html.parser')
return soup.find('h1').text
它的作用很简单，就是抓取指定的url，并且把标签为h1的元素提取出来
最后，我们新建文件task_dispatcher.py，内容如下
from workers import app
url_list = [
'http://docs.celeryproject.org/en/latest/getting-started/introduction.html',
'http://docs.celeryproject.org/en/latest/getting-started/brokers/index.html',
'http://docs.celeryproject.org/en/latest/getting-started/first-steps-with-celery.html',
'http://docs.celeryproject.org/en/latest/getting-started/next-steps.html',
'http://docs.celeryproject.org/en/latest/getting-started/resources.html',
'http://docs.celeryproject.org/en/latest/userguide/application.html',
'http://docs.celeryproject.org/en/latest/userguide/tasks.html',
'http://docs.celeryproject.org/en/latest/userguide/canvas.html',
'http://docs.celeryproject.org/en/latest/userguide/workers.html',
'http://docs.celeryproject.org/en/latest/userguide/daemonizing.html',
'http://docs.celeryproject.org/en/latest/userguide/periodic-tasks.html'
]
def manage_crawl_task(urls):
for url in urls:
app.send_task('tasks.crawl', args=(url,))
if __name__ == '__main__':
manage_crawl_task(url_list)
这段代码的作用主要就是给worker发送任务，任务是tasks.crawl，参数是url(元祖的形式)
现在，让我们在节点A(hostname为resolvewang的主机)上启动worker
celery -A workers worker -c 2 -l info
这里 -c指定了线程数为2， -l表示日志等级是info。我们把代码拷贝到节点B(节点名为wpm的主机)，同样以相同命令启动worker，便可以看到以下输出
两个节点
可以看到左边节点(A)先是all alone，表示只有一个节点；后来再节点B启动后，它便和B同步了
sync with celery@wpm
这个时候，我们运行给这两个worker节点发送抓取任务
python task_dispatcher.py
可以看到如下输出
分布式抓取示意图
可以看到两个节点都在执行抓取任务，并且它们的任务不会重复。我们再在redis里看看结果
backend示意图
可以看到一共有11条结果，说明 tasks.crawl中返回的数据都在db2(backend)中了，并且以json的形式存储了起来，除了返回的结果，还有执行是否成功等信息。
到此，我们就实现了一个很基础的分布式网络爬虫，但是它还不具有很好的扩展性，而且貌似太简单了…下一篇我将以微博数据采集为例来演示如何构建一个稳健的分布式网络爬虫。