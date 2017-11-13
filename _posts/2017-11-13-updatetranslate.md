---
layout: post
title: Android字串资源更新问题
---
## 问题描述

* 字符串资源翻译替换， 字符串有如下的格式

  ```xml
  <string name="vpn_no_ca_cert" msgid="2095005387500126113">"(ไม่ตรวจสอบเซิร์ฟเวอร์)"</string>
  ```

  ​

* 需要翻译的字符串保存在csv文件中， 可以用 name 与原始字符串对应.

  每一行包括name， 翻译后的字符串等内容.

## 方案策略

大体可以按照如下步骤完成任务：

* 读翻译后的字符串文件， 保存为dict 变量， key是string name， value是翻译后的字符串

* 读取原始string资源文件， 读得name字段， 判断name是否在dict中， 如果在则做字符串替换.

* 对上一步做处理之后， 写入新文件.

  新文件即最后包含翻译更新的结果.

## 涉及到的技术

* 正则表达式

  查找name字段等

  ```python
  import re
  def string_replace(str):
    pattern = re.compile(r'=".*?"')   # name="xxx", 非贪婪模式
    match = pattern.search(str)
    if (match):
    	key = match.group()[2:-1]
    	print ('key=', key)
      # 如果name在dt中， 即需要翻译， 则进行字符串拼接
    	if key in dt:
    	  pattern = re.compile(r'<.*?>')
    	  match = pattern.search(str)  	  
    	  return '    ' + match.group() + dt[key] + '</string>\n' 
    	else:
    	  return None
    else:
    	print str, 'return None'
    	return None
  ```

  ​

* pandas

  读取csv文件， 对内容进行处理， 比如保存为dict.

  ```python
  df = pd.read_csv('thai.csv')

  df_copy = df.where(df['owner']=='suqf')
  columns_to_keep = ['string-name', 'Thai-rev', 'owner']
  df_copy = df_copy[columns_to_keep]
  #print df_copy.head()
  #tuples = [df_copy['string-name']]

  dt = {}
  i = 0
  #构造dict
  while i < len(df_copy):
    dt[df_copy['string-name'][i]]=df_copy['Thai-rev'][i]
    i=i+1
  ```

  ​

* 文件读写

  ```python
  def read_file():
    file = 'strings.xml'
    with open(file, 'r') as f:
    	ft = f.readlines()
    return ft

  def write_back():
    file = 'strings_new.xml'
    with open(file, 'w') as f:
    	for line in result:
    	  newline=string_replace(line)
    	  if (newline == None):
    	  	f.write(line)
    	  else:
    	  	f.write(newline)
  ```

  ​

  ​

  ​



