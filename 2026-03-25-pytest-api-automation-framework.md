---
layout: page
title: 项目2
date: 2026-03-25
categories: 测试开发
tags: [Python, Pytest, 接口自动化, 测试开发, 实战]
---

# 基于Python+Pytest搭建支付接口自动化测试框架实战

在跨境支付业务的测试工作中，核心支付接口迭代频繁（每周2-3次），手工回归测试效率低、易漏测。因此我独立设计并搭建了一套基于 Python + Pytest 的接口自动化测试框架，本文分享完整实战过程。

---

## 一、框架设计目标
支付业务对**准确性、安全性、稳定性**要求极高，因此框架目标如下：

1. **高可维护性**：用例、数据、逻辑解耦，易于扩展
2. **高复用性**：通用请求、签名、日志统一封装
3. **高准确性**：接口响应 + 数据库流水双层断言
4. **易落地执行**：支持批量运行、可视化报告、CI/CD 集成

---

## 二、技术选型表

| 技术/工具 | 作用 |
| --- | --- |
| Python | 自动化开发语言 |
| Pytest | 测试用例管理与运行 |
| Requests | HTTP 接口请求 |
| YAML | 数据驱动，测试数据外置 |
| Allure | 生成美观测试报告 |

---

## 三、框架分层架构（五层设计）
采用**行业标准分层架构**，每层职责清晰，易维护、易扩展、易阅读。

---

### 1️⃣ 工具层封装（通用请求+日志+签名）
```python
# common/http_client.py
import requests
import logging
from common.sign_utils import generate_sign

logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')
logger = logging.getLogger(__name__)

class HttpClient:
    def __init__(self, base_url):
        self.base_url = base_url
        self.session = requests.Session()
        self.session.headers.update({
            "Content-Type": "application/json",
            "User-Agent": "AutoTest"
        })

    def request(self, method, url, **kwargs):
        full_url = self.base_url + url

        if "json" in kwargs:
            sign = generate_sign(kwargs["json"])
            self.session.headers["Sign"] = sign

        logger.info(f"请求: {method} {full_url}")
        try:
            response = self.session.request(method, full_url, timeout=10, **kwargs)
            logger.info(f"响应: {response.status_code}")
            return response
        except Exception as e:
            logger.error(f"请求失败: {e}")
            raise e

    def get(self, url, **kwargs):
        return self.request("GET", url, **kwargs)

    def post(self, url, **kwargs):
        return self.request("POST", url, **kwargs)
```

---

### 2️⃣ 接口层封装（业务 API 单独管理）
```python
# api/pay_api.py
from common.http_client import HttpClient

class PayApi(HttpClient):
    def __init__(self):
        super().__init__("https://api.wooshpay.com")

    # 支付下单接口
    def pay_order(self, order_data):
        return self.post("/v1/pay/order", json=order_data)

    # 订单查询接口
    def query_order(self, order_id):
        return self.get(f"/v1/pay/query/{order_id}")
```

---

### 3️⃣ 测试数据层（YAML 数据驱动）
```python
# testdata/pay_order_testdata.yaml
normal_case:
  - name: "正常支付-借记卡"
    order_data:
      order_id: "test_123456"
      amount: 100
      currency: "CNY"
      card_no: "6222021234567890"
    expected:
      status_code: 200
      code: 0
      status: "success"

abnormal_case:
  - name: "卡号错误"
    order_data:
      order_id: "test_789012"
      amount: 100
      card_no: "123456"
    expected:
      code: 40001
      message: "卡号格式错误"
```

---

### 4️⃣ 测试用例层（Pytest + Allure）
```python
# testcases/test_pay_order.py
import pytest
import allure
from api.pay_api import PayApi
from common.yaml_utils import read_yaml
from common.db_utils import DBUtils

test_data = read_yaml("testdata/pay_order_testdata.yaml")
pay_api = PayApi()
db = DBUtils()

@allure.feature("支付接口")
class TestPayOrder:

    @allure.title("正常支付场景：{case[name]}")
    @pytest.mark.parametrize("case", test_data["normal_case"])
    def test_normal_pay(self, case):
        # 发送请求
        resp = pay_api.pay_order(case["order_data"])
        res = resp.json()

        # 接口断言
        assert resp.status_code == 200
        assert res["code"] == 0

        # 数据库校验（支付核心！）
        order_id = case["order_data"]["order_id"]
        db_order = db.query_one(f"SELECT * FROM t_order WHERE order_id='{order_id}'")
        assert db_order is not None

    @allure.title("异常支付场景：{case[name]}")
    @pytest.mark.parametrize("case", test_data["abnormal_case"])
    def test_abnormal_pay(self, case):
        resp = pay_api.pay_order(case["order_data"])
        res = resp.json()
        assert res["code"] == case["expected"]["code"]
```

---

## 四、落地成果
这套框架上线后，取得了非常好的效果：效率大幅提升：核心支付接口实现 100% 场景覆盖，单次版本回归时间从原来的 4 个小时，压缩到了 15 分钟，测试效率提升 60% 以上。质量显著提高：通过接口 + 数据库的双层断言，在多次版本迭代中成功拦截了 3 起潜在的支付逻辑错误，有效规避了资金安全风险，上线版本零重大线上故障。维护成本降低：数据驱动的设计，让新增测试用例的时间从原来的 30 分钟 / 条，缩短到了 5 分钟 / 条，团队新人也能快速上手。可扩展性强：框架支持对接 Jenkins，实现代码提交后自动触发测试、自动生成测试报告，完美适配 CI/CD 流程。

---

## 五、经验总结
签名封装至关重要：支付接口的鉴权签名一定要封装好，不要在每个用例里写，不然签名逻辑改了，所有用例都要改，维护成本极高。数据库断言是核心：一定要加数据库断言，不要只依赖接口返回。接口返回成功不代表真的成功，只有数据库里的订单状态、资金流水正确，才是真的支付成功。避免硬编码等待：不要用固定的 sleep 等待，一定要用接口轮询或者事件等待机制，不然用例执行时间会很长，而且很容易因为网络波动导致用例失败。测试数据隔离：测试数据一定要做隔离，每次执行用例前要清理测试数据，避免上一次的用例执行结果影响下一次的用例。
