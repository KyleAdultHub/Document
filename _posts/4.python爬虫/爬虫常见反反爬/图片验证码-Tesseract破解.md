图片验证码-Tesseract破解
图像的翻译-Tesseract
文字识别
文字识别-机器视觉的一个分支，将图像翻译成文字一般被称为 光学文字识别(Optical Character Recognition, OCR)。可以实现OCR的底层库并不多,目前很多库都是使用共同的几个底层 OCR 库,或者是在上面 进行定制
ORC库的概念
在读取和处理图像、图像相关的机器学习以及创建图像等任务中，Python 一直都是非常出色的语言。虽然有很多库可以进行图像处理，但在这里我们只重点介绍： Tesseract
作用
Tesseract是一个将图像翻译成文字的OCR库(光学文字识别，Optical Character Recognition)，可以翻译图片中的文字内容，目前由 Google 赞助(Google 也是一家以 OCR 和机器学习技术闻名于世的公司)。Tesseract 是目前公认最优秀、最精确的开源 OCR 系统，除了极高的精确度，Tesseract 也具有很高的灵活性。它可以通过训练识别出任何字体，也可以识别出任何 Unicode 字符。
tesseract处理的文字规范
tesseract处理的文字大都是格式规范的，格式规范的文字具有以下特点:
使用一个标准字体(不包含手写体、草书,或者十分“花哨的”字体)
即使被复印或拍照，字体还是很清晰，没有多余的痕迹或污点
排列整齐，没有歪歪斜斜的字
没有超出图片范围，也没有残缺不全，或紧紧贴在图片的边缘
怎么将图片格式化
文字的一些格式问题在图片预处理时可以进行解决。例如,可以把图片转换成灰度图，调整亮度和对比度，还可以根据需要进行裁剪和旋转（详情需要了解图像与信号处理）等
使用步骤
安装tesseract和tensor-ocr
windows下安装
先下载tesseract-ocr
下载地址: https://digi.bib.uni-mannheim.de/tesseract
下载 pytenseract
pip install pytesseract pillow
ubuntu下安装
sudo apt-get install -y tesseract-ocr libtesseract-dev libleptonica-dev
pip install pytesseract pillow
centos下安装
yum install -y tesseract
pip install pytesseract pillow 
在终端中使用tesseract
tesseract  test.jpg  text      将翻译结果保存在text.txt中
tesseract chi_sim  text.png  paixu    将翻译结果保存在paixu.txt中（可以翻译中文简体）
tesseract --list-langs    查看当前支持的语言

在Python中使用tesesract
import pytesseract
from PIL import Image
image = Image.open(jpg)       获取图片
text = pytesseract.image_to_string(image, lang='chi_sim')       对图片的内容进行翻译
import subprocess
subprocess.call(['tessetact', '-1', 'chi_sim', filePath, 'paixu'])     使用终端命令进行翻译
pytesseract.file_to_text('image.png')       直接对图片进行识别
使用tesseract识别代码(包含图片处理)
import pytesseract
from PIL import Image
image = Image.open('code2.jpg')
# 图片灰度化处理
image = image.convert('L') 
# 图片二值化处理
threshold = 127        设置二值化阈值      
filter_func = lambda x: 0 if x < threshold else 1
image = image.point(filter_func, '1')
image.show()
result = pytesseract.image_to_text(image)     对处理后的图片进行识别
print(result)
对tesseract进行训练
训练的目的
tesseract一般只能识别符合文字规范的字体，其他的稍微复杂的字很难识别，tesseract可以通过训练对一种格式字体的反复训练，来实现字体的高精度识别
训练tesseract的方法
要训练 Tesseract 识别一种文字，无论是晦涩难懂的字体还是验证码，你都需要向 Tesseract 提供每个字符不同形式的样本。
首先要收集大量的验证码样本，样本的数量和复杂程度，会决定训练的效果。第二步是准确地告诉 Tesseract 一张图片中的每个字符是什么，以及每个字符的具体位置。
这里需要创建一些矩形定位文件(box file)，一个验证码图片生成一个矩形定位文件，也可以通过jTessBoxEditor软件来修改矩形的定位。
一个图片的矩形定位文件如下所示:

第一列符号是图片中的每个字符，后面的 4 个数字分别是包围这个字符的最小矩形的坐标 (图片左下角是原点 (0，0)，4 个数字分别对应每个字符的左下角 x 坐标、左下角 y 坐标、右上角 x 坐标和右上角 y 坐标)，最后一个数字“0”表示图片样本的编号。
矩形定位文件必须保存在一个 .box 后缀的文本文件中，(例如 4MmC3.box)。
tesseract训练教程
http://www.cnblogs.com/mjorcen/p/3800739.html?utm_source=tuicool&utm_medium=referral