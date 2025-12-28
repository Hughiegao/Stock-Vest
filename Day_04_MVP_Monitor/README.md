# 🐕 Day 04: 实时美股监控机器人 (MVP)

> **当前进度**：StockVest v2.0 - 最小可行性产品 (MVP)
> **功能描述**：一个 24 小时运行的自动监控流程。它每隔 30 分钟检查一次指定美股的价格，一旦触发预设阈值（如涨跌幅 > 1%），立即发送包含详细盘口数据的报警邮件。

---

## 🛑 避坑指南：为什么放弃 Yahoo Finance？

在开发初期，我们尝试使用 Yahoo Finance API，但遭遇了 **`401 Unauthorized`** 错误。
* **原因**：Yahoo 屏蔽了大多数云服务器厂商（如 Hostinger, AWS）的数据中心 IP。
* **解决方案**：紧急切换至 **[Finnhub.io](https://finnhub.io/)**。
    * ✅ **免费额度**：60 次请求 / 分钟（对监控来说绰绰有余）。
    * ✅ **稳定性**：专为开发者设计，JSON 结构极其标准。

---

## 🛠️ 架构设计 (Workflow Architecture)

本模块的 n8n 工作流由以下 4 个核心节点组成：

1.  **Schedule Trigger (闹钟)**:
    * 设置每 **30 分钟** 运行一次 (生产环境建议配置，避免邮件轰炸)。
2.  **Config (配置中心)**:
    * 使用 `Code Node` 统一管理全局变量（股票代码、阈值、API Key）。
    * **优势**：修改监控策略时无需触碰底层逻辑节点。
3.  **Finnhub API (眼睛)**:
    * `HTTP Request` 节点，动态读取 Config 中的参数抓取实时报价 (`Quote`).
4.  **Logic & Alert (大脑与嘴巴)**:
    * `If` 节点判断 `Math.abs(涨跌幅) > 阈值`。
    * `Send Email` 节点发送经过格式化处理的 HTML 邮件。

---

## ⚙️ 部署指南 (Setup Guide)

### 1. 准备工作
在导入代码前，请确保你拥有：
* **Finnhub API Key**: [点击注册免费版](https://finnhub.io/).
* **SMTP 服务**: 用于发邮件。推荐使用 Gmail 的 **App Password (应用专用密码)**。

### 2. 导入与配置
下载并导入本目录下的 `n8n_workflow.json` 后，请务必修改 **Config** 节点：

```javascript
// 示例：Config 节点代码
return {
  // 1. 监控目标
  symbol: "NVDA",
  
  // 2. 你的 Finnhub Key (必填)
  api_token: "sk_xxxxxxxxxxxxxx", 
  
  // 3. 报警阈值 (百分比)
  // 测试建议设为 1%，实战建议 3% - 5%
  thresholds: {
    "NVDA": 1,   
    "TSLA": 6
  }
}
```

### 3. 邮件模板优化 (关键代码)
为了解决计算机浮点数精度问题（如显示 `$120.333333`），我们在邮件模板中使用了 `.toFixed(2)` 方法。

**模板源码：**
```text
🚨 【StockVest 监控警报】: {{ $('Config').item.json.symbol }}

----------------------------------------
💰 当前价格: ${{ $('HTTP Request').item.json.c }}
📈 今日涨跌: {{ $('HTTP Request').item.json.dp.toFixed(2) }}% ( ${{ $('HTTP Request').item.json.d }} )
----------------------------------------

📊 市场广度:
• 今日最高 (High): ${{ $('HTTP Request').item.json.h }}
• 今日最低 (Low) : ${{ $('HTTP Request').item.json.l }}

💡 简单的分析:
当前价格距离今日最高点还差 ${{ ($('HTTP Request').item.json.h - $('HTTP Request').item.json.c).toFixed(2) }}
(如果差值很小，说明非常强势！)

----------------------------------------
🤖 触发阈值: {{ $('Config').item.json.thresholds[$('Config').item.json.symbol] }}%
⏰ 监控来源: Finnhub Real-time
```

---

## 📂 文件列表
* `n8n_workflow.json`: 完整的工作流 JSON 文件 (不含敏感信息)。
* `README.md`: 本说明文档。
