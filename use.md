# 代码简要分析

## yml测试用例

```
testinfo:
    - id: test004
      title: 每日新闻卡片浏览记录
      info: 打开知识
testcase:

    - element_info: com.huawei.works.knowledge:id/title
      find_type: ids
      index: 0
      operate_type: get_value
      info: 获取每日新闻下对第一条数据
    - element_info: com.huawei.works.knowledge:id/title
      find_type: ids
      index: 0
      operate_type: click
      is_time: 3
      info: 点击每日新闻下对第一条数据
    - element_info: h5-scroll
      find_type: id
      is_webview: 1 # 切换到webview
      info: 查找并获取详情页标题
    - element_info: com.huawei.works.knowledge:id/vtb_img_left
      find_type: id
      is_webview: 2 # 切换到native
      operate_type: click
      info: 点击返回按钮
    - element_info: com.huawei.works.knowledge:id/vtb_img_right2
      find_type: id
      operate_type: click
      info: 点击首页历史记录按钮
check:
    - element_info: com.huawei.works.knowledge:id/browser_knowledge_history_text
      find_type: ids
      index: 0
      operate_type: get_value
      check: ontrary_getval # 相反检查点，如删除后，此元素依然存在
      info: 查找是否存在历史记录
```


## 某个用例的page层

```
from PageObject import Pages


class DayNewHistoryPage:
    '''
    每日新闻浏览历史
    '''

    def __init__(self, kwargs):
        _init = {"driver": kwargs["driver"], "path": kwargs["path"], "device": kwargs["device"],
                 "logTest": kwargs["logTest"], "caseName": kwargs["caseName"]}
        self.page = Pages.PagesObjects(_init)

    def operate(self):  # 操作步骤
        self.page.operate()

    def checkPoint(self):  # 检查点
        self.page.checkPoint()
```

- pages再次封装了一层，主要可以看下重连机制的实现
  - 其实主要用的是launch_app+setupclass,另外一个好处是避免用例依赖，并不会重新启动一个session

```
  if result is not True and be.RE_CONNECT:
            self.msg = "用例失败重连过一次，失败原因:" + self.testInfo[0]["msg"]
            self.logTest.buildStartLine(self.caseName + "_失败重连")  # 记录日志
            self.operateElement.switchToNative()
            self.driver.launch_app()
            self.isOperate = True
            self.get_value = []
            self.is_get = False
            self.operate() # 执行步骤
            result = self.check(kwargs) # 坚持点
            self.testInfo[0]["msg"] = self.msg
        self.operateElement.switchToNative()
```

## testcase层调用page层

```
class HomeTest(ParametrizedTestCase):
    # 首页下拉
    def testAHomeSwipeDown(self):
        app = {}
        app["logTest"] = self.logTest
        app["driver"] = self.driver
        app["path"] = PATH("../yaml/home/HomeSwipeDown.yaml")
        app["device"] = self.devicesName
        app["caseName"] = sys._getframe().f_code.co_name

        page = HomeSwipeDownPage(app)
        page.operate()
        page.checkPoint()

    # banner浏览历史记录
    def testB0annerHistory(self):
        app = {}
        app["logTest"] = self.logTest
        app["driver"] = self.driver
        app["path"] = PATH("../yaml/home/BannerHistory.yaml")
        app["device"] = self.devicesName
        app["caseName"] = sys._getframe().f_code.co_name
        page = BannerHistoryPage(app)
        page.operate()
        page.checkPoint()

    @classmethod
    def setUpClass(cls):
        super(HomeTest, cls).setUpClass()

    @classmethod
    def tearDownClass(cls):
        super(HomeTest, cls).tearDownClass()
```

## 代码入口

```
def runnerCaseApp(devices):
    starttime = datetime.now()
    suite = unittest.TestSuite()
    suite.addTest(ParametrizedTestCase.parametrize(HomeTest, param=devices))
    suite.addTest(ParametrizedTestCase.parametrize(TestWeiQunTest, param=devices))
    unittest.TextTestRunner(verbosity=2).run(suite)
```


## 比较麻烦case处理
- 当遇到有些用例比较麻烦，必须单独写page 层，比如长按交换空间位置
-  自定义page层

```
.....
    def operate(self):
        for item in self.testCase:
            m_s_g = self.msg + "\n" if self.msg != "" else ""

            result = self.operateElement.operate(item, self.testInfo, self.logTest, self.device)
            if not result["result"]:
                msg = "执行过程中失败，请检查元素是否存在" + item["element_info"]
                print(msg)
                self.msg = m_s_g + msg
                self.testInfo[0]["msg"] = msg
                self.isOperate = False
                return False

            if item.get("operate_type", "0") == "location":
                app = {}
                web_element = self.driver.find_elements_by_id(item["element_info"])[item["index"]]
                start = web_element.location
                # 获取控件开始位置的坐标轴
                app["startX"] = start["x"]
                app["startY"] = start["y"]
                # 获取控件坐标轴差
                size1 = web_element.size

                width = size1["width"]
                height = size1["height"]
                # 计算出控件结束坐标
                endX = width + app["startX"]
                endY = height + app["startY"]

                app["endX"] = endX - 20
                app["endY"] = endY - 60

                self.location.append(app)
                # self.driver.swipe(endX, endY, starty, endY)
            if item.get("operate_type", "0") == be.GET_VALUE:
                self.get_value.append(result["text"])

            if item.get("is_swpie", "0") != "0":
                print(self.location)
                self.driver.swipe(self.location[0]["endX"], self.location[0]["endY"], self.location[1]["endX"], self.location[1]["endY"]+10)
```

- yaml用例
可以自定义一些关键字给page用

```
testinfo:
    - id: test019
      title: 拖动排序知识卡片
      info: 打开知识
testcase:
    - element_info: com.huawei.works.knowledge:id/vtb_img_right
      find_type: id
      operate_type: click
      info: 点击排序卡片按钮
    - element_info: com.huawei.works.knowledge:id/my_card_item_name_text
      find_type: ids
      index: 1
      operate_type: get_value
      info: 得到第二个卡片的值
    - element_info: com.huawei.works.knowledge:id/drag_item_image
      find_type: ids
      index: 0
      operate_type: location
      info: 得到第一卡片的坐标
    - element_info: com.huawei.works.knowledge:id/drag_item_image
      find_type: ids
      index: 1
      operate_type: location
      is_swpie: 1 # 特殊关键字，滑动指令
      info: 得到第二个卡片的坐标并拖动
    - element_info: com.huawei.works.knowledge:id/vtb_img_left
      find_type: id
      operate_type: click
      info: 点击返回按钮
check:
    - element_info: com.huawei.works.knowledge:id/title_txt
      find_type: id
      operate_type: get_value
```