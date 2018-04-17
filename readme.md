# 作者

作者：上海-悠悠
交流QQ群：588402570

# 前言
1.环境准备：
- python3.6
- requests
- xlrd
- openpyxl
- HTMLTestRunner_api

2.目前实现的功能：
- 封装requests请求方法
- 在excel填写接口请求参数
- 运行完后，重新生成一个excel报告，结果写入excel
- 用unittest+ddt数据驱动模式执行
- HTMLTestRunner生成可视化的html报告
- 对于没有关联的单个接口请求是可以批量执行的，需要登录的话写到setUpclass里的session里保持cookies
- token关联的不能实现
- logging日志文件暂时未加入

3.目前已知的缺陷：
- 无法实现参数关联：上个请求的结果是下个请求的参数，如token
- 接口请求参数名有重复的，目前未处理，如key1=value1&key1=value2,两个key都一样，这种需要用元组存储，目前暂时未判断
- 生成的excel样式未处理，后期慢慢优化样式
- python新手可能遇到模块导入报错问题

# 项目结构

![](https://images2018.cnblogs.com/blog/1070438/201803/1070438-20180323111753095-1203930062.png)




# excel测试数据

![](https://images2018.cnblogs.com/blog/1070438/201803/1070438-20180323102605467-2039444472.png)



# xlrd读excel数据

1.先从excel里面读取测试数据，返回字典格式

![](https://images2018.cnblogs.com/blog/1070438/201803/1070438-20180323102436666-427613061.png)


```
# coding:utf-8

# 作者：上海-悠悠
# QQ群：226296743

import xlrd
class ExcelUtil():
    def __init__(self, excelPath, sheetName="Sheet1"):
        self.data = xlrd.open_workbook(excelPath)
        self.table = self.data.sheet_by_name(sheetName)
        # 获取第一行作为key值
        self.keys = self.table.row_values(0)
        # 获取总行数
        self.rowNum = self.table.nrows
        # 获取总列数
        self.colNum = self.table.ncols

    def dict_data(self):
        if self.rowNum <= 1:
            print("总行数小于1")
        else:
            r = []
            j = 1
            for i in list(range(self.rowNum-1)):
                s = {}
                # 从第二行取对应values值
                s['rowNum'] = i+2
                values = self.table.row_values(j)
                for x in list(range(self.colNum)):
                    s[self.keys[x]] = values[x]
                r.append(s)
                j += 1
            return r

if __name__ == "__main__":
    filepath = "debug_api.xlsx"
    sheetName = "Sheet1"
    data = ExcelUtil(filepath, sheetName)
    print(data.dict_data())
```
# openpyxl写入数据


1.再封装一个写入excel数据的方法

```
# coding:utf-8
from openpyxl import load_workbook
import openpyxl

# 作者：上海-悠悠
# QQ群：226296743

def copy_excel(excelpath1, excelpath2):
    '''复制excek，把excelpath1数据复制到excelpath2'''
    wb2 = openpyxl.Workbook()
    wb2.save(excelpath2)
    # 读取数据
    wb1 = openpyxl.load_workbook(excelpath1)
    wb2 = openpyxl.load_workbook(excelpath2)
    sheets1 = wb1.sheetnames
    sheets2 = wb2.sheetnames
    sheet1 = wb1[sheets1[0]]
    sheet2 = wb2[sheets2[0]]
    max_row = sheet1.max_row         # 最大行数
    max_column = sheet1.max_column   # 最大列数

    for m in list(range(1,max_row+1)):
        for n in list(range(97,97+max_column)):   # chr(97)='a'
            n = chr(n)                            # ASCII字符
            i ='%s%d'% (n, m)                     # 单元格编号
            cell1 = sheet1[i].value               # 获取data单元格数据
            sheet2[i].value = cell1               # 赋值到test单元格

    wb2.save(excelpath2)                 # 保存数据
    wb1.close()                          # 关闭excel
    wb2.close()

class Write_excel(object):
    '''修改excel数据'''
    def __init__(self, filename):
        self.filename = filename
        self.wb = load_workbook(self.filename)
        self.ws = self.wb.active  # 激活sheet

    def write(self, row_n, col_n, value):
        '''写入数据，如(2,3，"hello"),第二行第三列写入数据"hello"'''
        self.ws.cell(row_n, col_n).value = value
        self.wb.save(self.filename)

if __name__ == "__main__":
    copy_excel("debug_api.xlsx", "testreport.xlsx")
    wt = Write_excel("testreport.xlsx")
    wt.write(4, 5, "HELLEOP")
    wt.write(4, 6, "HELLEOP")

```

# 封装request请求方法

1.把从excel读处理的数据作为请求参数，封装requests请求方法，传入请求参数，并返回结果

2.为了不污染测试的数据，出报告的时候先将测试的excel复制都应该新的excel

3.把测试返回的结果，在新的excel里面写入数据
```
# coding:utf-8
import json
import requests
from excelddtdriver.common.readexcel import ExcelUtil
from excelddtdriver.common.writeexcel import copy_excel, Write_excel

# 作者：上海-悠悠
# QQ群：226296743


def send_requests(s, testdata):
    '''封装requests请求'''
    method = testdata["method"]
    url = testdata["url"]
    # url后面的params参数
    try:
        params = eval(testdata["params"])
    except:
        params = None
    # 请求头部headers
    try:
        headers = eval(testdata["headers"])
        print("请求头部：%s" % headers)
    except:
        headers = None
    # post请求body类型
    type = testdata["type"]

    test_nub = testdata['id']
    print("*******正在执行用例：-----  %s  ----**********" % test_nub)
    print("请求方式：%s, 请求url:%s" % (method, url))
    print("请求params：%s" % params)

    # post请求body内容
    try:
        bodydata = eval(testdata["body"])
    except:
        bodydata = {}

    # 判断传data数据还是json
    if type == "data":
        body = bodydata
    elif type == "json":
        body = json.dumps(bodydata)
    else:
        body = bodydata
    if method == "post": print("post请求body类型为：%s ,body内容为：%s" % (type, body))

    verify = False
    res = {}   # 接受返回数据

    try:
        r = s.request(method=method,
                      url=url,
                      params=params,
                      headers=headers,
                      data=body,
                      verify=verify
                       )
        print("页面返回信息：%s" % r.content.decode("utf-8"))
        res['id'] = testdata['id']
        res['rowNum'] = testdata['rowNum']
        res["statuscode"] = str(r.status_code)  # 状态码转成str
        res["text"] = r.content.decode("utf-8")
        res["times"] = str(r.elapsed.total_seconds())   # 接口请求时间转str
        if res["statuscode"] != "200":
            res["error"] = res["text"]
        else:
            res["error"] = ""
        res["msg"] = ""
        if testdata["checkpoint"] in res["text"]:
            res["result"] = "pass"
            print("用例测试结果:   %s---->%s" % (test_nub, res["result"]))
        else:
            res["result"] = "fail"
        return res
    except Exception as msg:
        res["msg"] = str(msg)
        return res

def wirte_result(result, filename="result.xlsx"):
    # 返回结果的行数row_nub
    row_nub = result['rowNum']
    # 写入statuscode
    wt = Write_excel(filename)
    wt.write(row_nub, 8, result['statuscode'])       # 写入返回状态码statuscode,第8列
    wt.write(row_nub, 9, result['times'])            # 耗时
    wt.write(row_nub, 10, result['error'])            # 状态码非200时的返回信息
    wt.write(row_nub, 12, result['result'])           # 测试结果 pass 还是fail
    wt.write(row_nub, 13, result['msg'])           # 抛异常

if __name__ == "__main__":
    data = ExcelUtil("debug_api.xlsx").dict_data()
    print(data[0])
    s = requests.session()
    res = send_requests(s, data[0])
    copy_excel("debug_api.xlsx", "result.xlsx")
    wirte_result(res, filename="result.xlsx")
```

# 测试用例unittest+ddt

1.测试用例用unittest框架组建，并用ddt数据驱动模式，批量执行用例

```
# coding:utf-8
import unittest
import ddt
import os
import requests
from excelddtdriver.common import base_api
from excelddtdriver.common import readexcel
from excelddtdriver.common import writeexcel

# 作者：上海-悠悠
# QQ群：226296743

# 获取demo_api.xlsx路径
curpath = os.path.dirname(os.path.realpath(__file__))
testxlsx = os.path.join(curpath, "demo_api.xlsx")

# 复制demo_api.xlsx文件到report下
report_path = os.path.join(os.path.dirname(curpath), "report")
reportxlsx = os.path.join(report_path, "result.xlsx")

testdata = readexcel.ExcelUtil(testxlsx).dict_data()
@ddt.ddt
class Test_api(unittest.TestCase):
    @classmethod
    def setUpClass(cls):
        cls.s = requests.session()
        # 如果有登录的话，就在这里先登录了
        writeexcel.copy_excel(testxlsx, reportxlsx) # 复制xlsx

    @ddt.data(*testdata)
    def test_api(self, data):
        # 先复制excel数据到report
        res = base_api.send_requests(self.s, data)

        base_api.wirte_result(res, filename=reportxlsx)
        # 检查点 checkpoint
        check = data["checkpoint"]
        print("检查点->：%s"%check)
        # 返回结果
        res_text = res["text"]
        print("返回实际结果->：%s"%res_text)
        # 断言
        self.assertTrue(check in res_text)

if __name__ == "__main__":
    unittest.main()
```

# 生成报告

1.用HTMLTestRunner生成html报告，我这里改了下名称，改成了HTMLTestRunner_api.py
此文件跟selenium的报告是通用的，github可下载[https://github.com/yoyoketang/selenium_report/tree/master/selenium_report](https://github.com/yoyoketang/selenium_report/tree/master/selenium_report)

```
# coding=utf-8
import unittest
import time
from excelddtdriver.common import HTMLTestRunner_api
import os

# 作者：上海-悠悠
# QQ群：226296743

curpath = os.path.dirname(os.path.realpath(__file__))
report_path = os.path.join(curpath, "report")
if not os.path.exists(report_path): os.mkdir(report_path)
case_path = os.path.join(curpath, "case")

def add_case(casepath=case_path, rule="test*.py"):
    '''加载所有的测试用例'''
    # 定义discover方法的参数
    discover = unittest.defaultTestLoader.discover(casepath,
                                                  pattern=rule,)

    return discover

def run_case(all_case, reportpath=report_path):
    '''执行所有的用例, 并把结果写入测试报告'''
    htmlreport = reportpath+r"\result.html"
    print("测试报告生成地址：%s"% htmlreport)
    fp = open(htmlreport, "wb")
    runner = HTMLTestRunner_api.HTMLTestRunner(stream=fp,
                                               verbosity=2,
                                               title="测试报告",
                                               description="用例执行情况")

    # 调用add_case函数返回值
    runner.run(all_case)
    fp.close()

if __name__ == "__main__":
    cases = add_case()
    run_case(cases)

```

2.生成的excel报告

![](https://images2018.cnblogs.com/blog/1070438/201803/1070438-20180323102540571-718340736.png)


3.生成的html报告

![](https://images2018.cnblogs.com/blog/1070438/201803/1070438-20180323102617264-2071969634.png)



---------------------------------python接口自动化已出书-------------------------
买了此书的小伙伴可以在书的最后一篇下载到源码

全书购买地址 [https://yuedu.baidu.com/ebook/585ab168302b3169a45177232f60ddccda38e695](https://yuedu.baidu.com/ebook/585ab168302b3169a45177232f60ddccda38e695)
![](https://images2018.cnblogs.com/blog/1070438/201803/1070438-20180323104725561-146885286.png)