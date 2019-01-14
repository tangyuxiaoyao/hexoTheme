---
title: selenium 爬取淘宝商品明细数据
date: 2017-12-21 16:11:44
tags: [selenium,python,爬虫]
---

## 需求背景:
>selenium 爬取淘宝商品明细数据<!--more-->


## 技术分析
>确定元素的有针对性检索:
①：区分输入框和带有点击相应事件的点击按钮(**div>p**	选择父元素为 **div** 元素的所有 **p**元素。)
②：如果是搜索的是多级样式选择器（**div p** 选择 **div** 元素内部的所有**p** 元素。），这种一般会拿到多个元素返回，的确也是我们需要的多个商品item, 确保这些元素存在以后，然后进一步将这些dom树使用pyquery(python+jquery)解析
ps : presence_of_element_located : returns the WebElement once it is located


## 技术实现
>config.py

``` python
MONGO_URL = 'localhost'
MONGO_DB = 'taobao'
MONGO_TABLE = 'productions'

SERVER_ARGS = ['--disk-cache=true', '--load-images=false']

```

>tbPage.py
``` python
# -*-coding:utf-8-*-

import sys

from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.support import expected_conditions as EC
from selenium.webdriver.support.ui import WebDriverWait
from selenium.common.exceptions import TimeoutException
import re
from pyquery import PyQuery as pq
import json
from mongConfig import *
import pymongo

reload(sys)
sys.setdefaultencoding('utf-8')

# 启动浏览器模拟操作
# browser = webdriver.Chrome()
# 模拟浏览器环境
browser = webdriver.PhantomJS(service_args=SERVER_ARGS)
# 模拟浏览器窗口大小
browser.set_window_size("1400", '900')
wait = WebDriverWait(browser, 10)
browser.get("https://www.taobao.com")

# 初始化mongo链接

mongoClient = pymongo.MongoClient(MONGO_URL)
mongoDB = mongoClient[MONGO_DB]


# 确定元素的有针对性检索:
# ①：区分输入框和带有点击相应事件的点击按钮(div>p	选择父元素为 <div> 元素的所有 <p> 元素。)
# ②：如果是搜索的是多级样式选择器（div p 选择 <div> 元素内部的所有 <p> 元素。），这种一般会拿到多个元素返回，的确也是我们需要的多个商品item
# 这种需要进一步将这些dom树使用pyquery(python+jquery)解析，
# presence_of_element_located : returns the WebElement once it is located
def search(keyword):
    try:
        input = wait.until(
            EC.presence_of_element_located((By.CSS_SELECTOR, "#q"))
        )

        commit = wait.until(
            EC.element_to_be_clickable((By.CSS_SELECTOR, "#J_TSearchForm > div.search-button > button"))
        )

        # 输入框填充搜素关键词
        input.send_keys(keyword.decode('utf-8'))

        # 模拟点击搜素按钮→触发搜素
        commit.click()

        page_total = wait.until(
            EC.presence_of_element_located((By.CSS_SELECTOR, "#mainsrp-pager > div > div > div > div.total"))
        )
        # 首页不参与分页翻页的逻辑
        parse_content()
        return page_total.text
    except TimeoutException:
        return search(keyword)


def next_page(pagenum):
    print "正在采集第%d页数据" % pagenum
    now_page = wait.until(
        EC.presence_of_element_located((By.CSS_SELECTOR, "#mainsrp-pager > div > div > div > div.form > input"))
    )

    next = wait.until(
        EC.element_to_be_clickable((By.CSS_SELECTOR, "#mainsrp-pager > div > div > div > div.form > span.btn.J_Submit"))
    )

    now_page.clear()
    now_page.send_keys(pagenum)
    next.click()

    # 判断成功的前提是高亮的分页数字为传入的分页页码
    WebDriverWait(browser, 100).until(
        EC.text_to_be_present_in_element(
            (By.CSS_SELECTOR, "#mainsrp-pager > div > div > div > ul > li.item.active > span"), str(pagenum))
    )


def parse_content():
    try:
        wait.until(EC.presence_of_element_located((By.CSS_SELECTOR, "#mainsrp-itemlist .items .item")))

        content_html = browser.page_source
        context = pq(content_html)
        items = context('#mainsrp-itemlist .items .item').items()
        for item in items:
            product = {
                'image': item.find('.pic .img').attr('src'),
                'price': item.find('.price').text(),
                'deal-cnt': item.find('.deal-cnt').text()[:-3],
                'title': item.find('.title').text(),
                'shop': item.find('.shop').text(),
                'location': item.find('.location').text()
            }
            utf_product = json.dumps(product, ensure_ascii=False)
            print utf_product
            # save_to_mongo(product)


    except TimeoutException:
        parse_content()


def save_to_mongo(product):
    utf_title = json.dumps(product['title'], ensure_ascii=False)
    if mongoDB[MONGO_TABLE].insert(product):
        print  "%s\t插入mongo成功" % utf_title
    else:
        print "%s存储到mongo失败" % utf_title


def page_query(keyword):
    try:
        total = search(keyword)
        total = int(re.compile("(\\d+)").search(total).group(1))
        for i in range(2, total + 1):
            next_page(i)
            parse_content()

    finally:
        browser.close()


if __name__ == '__main__':
    keyword = "零食"
    page_query(keyword)

```

## 注意点

① 发起请求要求的必须是unicode码,否则会报错('utf8' codec can't decode byte 0xe9 in position 0: unexpected end of data), input.send_keys(keyword.decode('utf-8'))
>   详情参考selenium utils.py 中的如下代码:

``` python
def keys_to_typing(value):
    """Processes the values that will be typed in the element."""
    typing = []
    for val in value:
        if isinstance(val, Keys):
            typing.append(val)
        elif isinstance(val, int):
            val = str(val)
            for i in range(len(val)):
                typing.append(val[i])
        else:
            for i in range(len(val)):
                typing.append(val[i])
    return typing
```

>上面代码中提到的Keys的部分代码
``` python
class Keys(object):
    """
    Set of special keys codes.
    """

    NULL = '\ue000'
    CANCEL = '\ue001'  # ^break
    HELP = '\ue002'
    BACKSPACE = '\ue003'
    BACK_SPACE = BACKSPACE
    TAB = '\ue004'
    CLEAR = '\ue005'
```


 ②   判断成功的前提是高亮的分页数字为传入的分页页码


``` python
    WebDriverWait(browser, 100).until(
        EC.text_to_be_present_in_element(
            (By.CSS_SELECTOR, "#mainsrp-pager > div > div > div > ul > li.item.active > span"), str(pagenum))
    )
```
>这里拿到传入的页数以后要转换为str，否则会报错如下：TypeError: coercing to Unicode: need string or buffer, int found
>究其原因如下：

``` python
        class text_to_be_present_in_element(object):
    """ An expectation for checking if the given text is present in the
    specified element.
    locator, text
    """
    def __init__(self, locator, text_):
        self.locator = locator
        self.text = text_

    def __call__(self, driver):
        try:
            element_text = _find_element(driver, self.locator).text
			&nbsp; &nbsp; #重点看这里，传入的值非String类型就会导致报错。
            return self.text in element_text
        except StaleElementReferenceException:
            return False
```
