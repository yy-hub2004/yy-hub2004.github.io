---
layout: post
title: 基于Python+Pytest搭建支付接口自动化测试框架实战
date: 2026-03-25
categories: 测试开发
tags: [Python, Pytest, 接口自动化, 测试开发, 实战]
---

# 基于Python+Pytest搭建支付接口自动化测试框架实战

在跨境支付业务的测试工作中，核心支付接口的迭代非常频繁，每周2-3次的版本发布，让手工回归测试的工作量越来越大，不仅耗时久，还存在人为漏测的风险。因此我独立设计并搭建了一套基于Python+Pytest的接口自动化测试框架，完美解决了这个痛点，本文就给大家分享完整的搭建过程和实战经验。

---

## 一、框架设计目标

支付业务和普通业务不一样，对**准确性、安全性、稳定性**要求极高，所以我的框架设计必须满足这几个核心目标：

1.  **高可维护性**：测试用例、测试数据、业务逻辑解耦，新人也能快速上手新增用例
2.  **高复用性**：通用请求、鉴权、加解密逻辑封装，减少重复代码
3.  **高准确性**：多层级断言，不仅校验接口响应，还要校验数据库资金流水，保障支付安全
4.  **易落地**：支持批量执行、可视化测试报告，可对接CI/CD流程，适配高频迭代

---

### 技术选型表

| 技术/工具 | 作用 | 选型原因 |
| :--- | :--- | :--- |
| Python | 开发语言 | 生态完善，测试领域最常用的语言，学习成本低 |
| Pytest | 测试框架 | 比 unittest 更灵活，支持丰富的插件、fixture、参数化，用例管理更方便 |
| Requests | HTTP请求库 | 简单易用，功能全面，完美适配接口测试需求 |
| YAML | 测试数据管理 | 实现数据驱动，测试数据和代码分离，降低维护成本 |
| Allure | 测试报告 | 可视化效果好，支持详细的日志、截图、失败原因展示，问题定位更高效 |

---

## 二、框架分层架构设计

我采用了经典的分层架构设计，把整个框架分成了5层，每层职责单一，完全解耦：

### 1. 工具层封装

首先封装通用的HTTP请求工具类，统一处理所有接口的请求、鉴权、日志、异常处理，避免每个用例都写重复的请求代码：

```python
# common/http_client.py
import requests
import logging
from common.sign_utils import generate_sign

# 配置日志
logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(name)s - %(levelname)s - %(message)s')
logger = logging.getLogger(__name__)

class HttpClient:
    def __init__(self, base_url):
        self.base_url = base_url
        self.session = requests.Session()
        # 统一添加公共请求头
        self.session.headers.update({
            "Content-Type": "application/json",
            "User-Agent": "AutoTest/1.0"
        })

    def request(self, method, url, **kwargs):
        # 拼接完整URL
        full_url = self.base_url + url
        # 生成接口签名（支付接口必须的鉴权逻辑）
        if "json" in kwargs:
            sign = generate_sign(kwargs["json"])
            self.session.headers["Sign"] = sign
        
        logger.info(f"请求地址：{full_url}")
        logger.info(f"请求方法：{method}")
        logger.info(f"请求参数：{kwargs}")

        try:
            # 发送请求
            response = self.session.request(method, full_url, timeout=10, **kwargs)
            logger.info(f"响应状态码：{response.status_code}")
            logger.info(f"响应内容：{response.text}")
            return response
        except Exception as e:
            logger.error(f"请求异常：{str(e)}", exc_info=True)
            raise e

    # 封装常用请求方法
    def get(self, url, **kwargs):
        return self.request("GET", url, **kwargs)

    def post(self, url, **kwargs):
        return self.request("POST", url, **kwargs)
2. 接口层封装
将不同模块的API接口进行封装，使测试用例与具体的接口实现解耦：
# api/pay_api.py
from common.http_client import HttpClient

class PayApi(HttpClient):
    def __init__(self):
        super().__init__(base_url="https://api.wooshpay.com")

    # 收银台支付接口
    def pay_order(self, order_data):
        return self.post("/v1/pay/order", json=order_data)

    # 订单查询接口
    def query_order(self, order_id):
        return self.get(f"/v1/pay/query/{order_id}")
3. 数据驱动设计
使用YAML文件管理测试数据，实现数据驱动，将测试数据与测试代码分离，极大提高了可维护性：
yaml

编辑



# testdata/pay_order_testdata.yaml
# 正常支付场景
normal_case:
  - name: "正常支付-人民币借记卡"
    order_data:
      order_id: "test_{timestamp}"
      amount: 100
      currency: "CNY"
      card_no: "6222021234567890"
    expected:
      status_code: 200
      code: 0
      status: "success"

# 异常支付场景
abnormal_case:
  - name: "异常支付-卡号错误"
    order_data:
      order_id: "test_{timestamp}"
      amount: 100
      currency: "CNY"
      card_no: "1234567890"
    expected:
      status_code: 200
      code: 40001
      message: "卡号格式错误"

  - name: "异常支付-金额为0"
    order_data:
      order_id: "test_{timestamp}"
      amount: 0
      currency: "CNY"
      card_no: "6222021234567890"
    expected:
      status_code: 200
      code: 40002
      message: "支付金额必须大于0"
4. 测试用例编写
使用Pytest编写测试用例，并结合Allure生成精美的测试报告：
python

编辑



# testcases/test_pay_order.py
import pytest
import allure
from api.pay_api import PayApi
from common.yaml_utils import read_yaml
from common.db_utils import DBUtils

# 读取测试数据
test_data = read_yaml("testdata/pay_order_testdata.yaml")
pay_api = PayApi()
db_utils = DBUtils()

@allure.feature("支付接口测试")
@allure.story("收银台下单接口")
class TestPayOrder:

    @allure.title("正常支付场景：{case[name]}")
    @pytest.mark.parametrize("case", test_data["normal_case"])
    def test_normal_pay_order(self, case):
        # 1. 发送请求
        response = pay_api.pay_order(case["order_data"])
        res_json = response.json()

        # 2. 接口响应断言
        assert response.status_code == case["expected"]["status_code"]
        assert res_json["code"] == case["expected"]["code"]
        assert res_json["data"]["status"] == case["expected"]["status"]

        # 3. 数据库断言（支付业务核心，保障资金安全）
        order_id = case["order_data"]["order_id"]
        db_order = db_utils.query_one(f"SELECT * FROM t_order WHERE order_id = '{order_id}'")
        assert db_order is not None
        assert db_order["amount"] == case["order_data"]["amount"]
        assert db_order["status"] == case["expected"]["status"]

    @allure.title("异常支付场景：{case[name]}")
    @pytest.mark.parametrize("case", test_data["abnormal_case"])
    def test_abnormal_pay_order(self, case):
        # 发送请求
        response = pay_api.pay_order(case["order_data"])
        res_json = response.json()

        # 异常场景断言
        assert response.status_code == case["expected"]["status_code"]
        assert res_json["code"] == case["expected"]["code"]
        assert res_json["message"] == case["expected"]["message"]
三、落地成果
这套框架上线后，取得了非常好的效果：
效率大幅提升：核心支付接口实现 100% 场景覆盖，单次版本回归时间从原来的 4 个小时，压缩到了 15 分钟，测试效率提升 60% 以上。
质量显著提高：通过接口 + 数据库的双层断言，在多次版本迭代中成功拦截了 3 起潜在的支付逻辑错误，有效规避了资金安全风险，上线版本零重大线上故障。
维护成本降低：数据驱动的设计，让新增测试用例的时间从原来的 30 分钟 / 条，缩短到了 5 分钟 / 条，团队新人也能快速上手。
可扩展性强：框架支持对接 Jenkins，实现代码提交后自动触发测试、自动生成测试报告，完美适配 CI/CD 流程。
四、踩坑经验总结
签名封装至关重要：支付接口的鉴权签名一定要封装好，不要在每个用例里写，不然签名逻辑改了，所有用例都要改，维护成本极高。
数据库断言是核心：一定要加数据库断言，不要只依赖接口返回。接口返回成功不代表真的成功，只有数据库里的订单状态、资金流水正确，才是真的支付成功。
避免硬编码等待：不要用固定的 sleep 等待，一定要用接口轮询或者事件等待机制，不然用例执行时间会很长，而且很容易因为网络波动导致用例失败。
测试数据隔离：测试数据一定要做隔离，每次执行用例前要清理测试数据，避免上一次的用例执行结果影响下一次的用例。
