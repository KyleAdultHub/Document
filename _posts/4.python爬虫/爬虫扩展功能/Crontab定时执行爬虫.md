Crontab的使用方法
安装cron软件
apt-get  install  cron
编辑crontab定时执行命令
进入crontab编辑界面
crontab  -e    进入编辑界面
crontab  -l     查看当前的定时任务
crontab -r    删除任务
编辑需要被定时执行的命令
编辑的格式
分(0-59)   小时(0-23)  日(1-31)   月(1-12)  星期(0-6)  命令(command)
示例
30  7  8  *  *  ls    指定每月的8日的7:30执行ls命令
*/15  *  *  *  *  ls      每15分钟执行一次ls命令
0  */2  *  *  *  ls     每隔两个小时执行一次ls
注意点
* /num 代表每隔多长时间的意思
  当一个位置使用每隔符号的时候，其前边的时间位置，不能为*
  星期中0表示周日
  使用Crontab定时爬虫
  编辑python_spider命令
  先把python的执行命令写入.sh脚本
  #！/bin/sh     将脚本定义在可执行脚本目录中
  cd ` dirname` $0 || exit 1    cd到.sh文件所在目录(项目目录)，失败则退出，dirname两边的不是引号
  所以我们要将.sh文件定义在项目目录中，执行的时候，会自动的执行cd命令
  topython3    切换到python3的环境
  scrapy crawl spider_name >> run.log 2>&1    
  将终端显示的内容重定向到log日志中，不会在终端显示错误、异常（2>&1，表示会将错误和异常也保存到日志中  ）
  给.sh文件添加可执行权限
  chmod  +  run.sh
  在crontab中编辑脚本文件执行时间（注意些绝对路径）
  0  6   *  *  *  /home/ubuntu/***/myspider.sh >> /home/ubuntu/*** run_crontab.log 2>&1
  将终端显示的内容重定向到log日志中，不会保存错误、异常（2>&1，表示会将错误和异常也保存到日志中  ）
  动态查看log日志
  tail  -f  log_name.log