---
layout: post
title: 基于Python+Pytest搭建支付接口自动化测试框架实战
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

# 三、框架分层架构（五层设计）
我采用**行业标准分层架构**，每层职责清晰，易维护、易扩展、易阅读。

---

# 1️⃣ 工具层封装（通用请求+日志+签名）
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
2️⃣ 接口层封装（业务 API 单独管理）
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
