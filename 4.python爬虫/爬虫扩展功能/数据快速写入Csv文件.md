csv文件的写入
以列表的方式写入
import  csv      导入csv模块
with open('data.csv', 'w', encoding='utf-8') as csvfile:
writer = csv.writer(csvfile, delimiter=',')    创建阅读器
writer.writerow(['id', 'name', 'age'])    向csv文件中写入一行数据
writer.writerows([ ['1', '2', '3'], ['2', '3', '4'] ])     向csv文件中写入多行数据
以字典的方式写入
import  csv      导入csv模块
with open('data.csv', 'w', encoding='utf-8') as csvfile:
fieldnames = ['id', 'name', 'age']
writer = csv.DictWriter(csvfile, fieldnames=fieldnames)    创建阅读器， 并指定列索引
writer.writeheader()      向文件中写入列的索引
writer.writerow({'id': 1, 'name': 2, 'age': 3})    向csv文件中写入一行数据
csv文件的读取
csv文件按行读取
import csv
with open('data.csv', 'r', encoding='utf-8') as csvfile:
reader = csv.reader(csvfile)      创建阅读器
for row in reader:
print(row)
>> ['id', 'name', 'age']
>> 使用pandas进行csv文件的写入和读取
>> 读取csv文件数据的方法
>> df_obj = pd.read_csv('filename', [usecols=['col1', 'col2'], skiprows=n1, skipfooter=n2, index_col=n3, engine='python'])     返回的是DataFrame对象
>> filename：表示读取的文件名
>> usecols：表示读取文件的哪些列
>> skiprows:  表示跳过文件的前几行
>> skipfooter： 表示跳过文件的后几行
>> index_col：表示使用那一列作为索引列
>> engine： 使用的解释器引擎， 当读取中文文件的时候需要指定
>> 写入csv数据的方法
>> df_obj.to_csv("filename")
>> filename：表示读取的文件名,要加上.csv后缀；