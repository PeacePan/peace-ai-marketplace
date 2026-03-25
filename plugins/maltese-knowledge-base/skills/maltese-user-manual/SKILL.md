---
name: maltese-user-manual
description: 用於回答 Maltese POS 系統終端使用者的操作問題，涵蓋一般商品結帳、美容商品結帳、美容工作臺三個入口的 UI 流程說明
---

# Maltese POS 使用手冊

## 系統概覽

Maltese 是寵物店 POS 系統，支援三種操作介面：一般商品結帳、美容商品結帳、美容工作臺。所有介面都從同一個登入與機台選擇畫面啟動，各介面針對不同業務需求設計。

## 三個入口快速對照表

| 入口 | 網址 | 使用情境 | Tab 數量 |
|------|------|---------|---------|
| 一般商品結帳 | `/checkout` | 銷售寵物實體商品 | 3 個：結帳、商品、管理 |
| 美容商品結帳 | `/salon` | 銷售美容服務 | 2 個：結帳、管理 |
| 美容工作臺 | `/workbench` | 為工單指派美容師 | 無 Tab，單一看板 |

## 開機流程

三個入口共用相同的開機與登入流程，詳見：`entries/product/references/01-boot-up-and-login.md`

## 各入口詳細說明

### 一般商品結帳

- Agent 指引：`entries/product/AGENT.md`
- 操作流程：
  - `references/01-boot-up-and-login.md` — 開機與登入
  - `references/02-open-register.md` — 開帳
  - `references/03-search-and-add-items.md` — 搜尋與加入商品
  - `references/04-set-member.md` — 設定會員
  - `references/05-apply-promotions-and-discounts.md` — 促銷與折扣
  - `references/06-stash-and-resume-sale.md` — 暫結與取回
  - `references/07-checkout-and-payment.md` — 小計與付款
  - `references/08-invoice-setup.md` — 發票設定
  - `references/09-exchange-sale.md` — 換購銷售
  - `references/10-full-and-partial-return.md` — 退款
  - `references/11-delivery-order-import.md` — 外送訂單帶入
  - `references/12-voucher-and-deposit.md` — 票券與保證金
  - `references/13-reports-and-management.md` — 報表與管理
  - `references/14-close-register.md` — 清帳

### 美容商品結帳

- Agent 指引：`entries/salon/AGENT.md`
- 操作流程：
  - `references/01-boot-up-salon.md` — 開機（美容入口）
  - `references/02-open-register.md` — 開帳
  - `references/03-set-member-and-pets.md` — 設定會員與寵物
  - `references/04-add-or-edit-service.md` — 新增或編輯服務
  - `references/05-apply-salon-discounts.md` — 美容折扣
  - `references/06-pay-later-flow.md` — 後付款
  - `references/07-checkout-and-payment-salon.md` — 小計與付款
  - `references/08-worksheet-and-service-search.md` — 工單與銷售查詢
  - `references/09-full-and-package-return.md` — 退款
  - `references/10-cancel-sale.md` — 取消銷售
  - `references/11-invoice-and-clear.md` — 發票與清除
  - `references/12-reports-and-management-salon.md` — 報表與管理

### 美容工作臺

- Agent 指引：`entries/workbench/AGENT.md`
- 操作流程：
  - `references/01-boot-up-workbench.md` — 開機（工作臺）
  - `references/02-workbench-overview.md` — 介面總覽
  - `references/03-read-work-orders.md` — 閱讀工單
  - `references/04-assign-beautician.md` — 指派美容師
  - `references/05-change-beautician-on-task.md` — 變更指派
  - `references/06-save-assignments.md` — 儲存指派結果
  - `references/07-refresh-and-date-query.md` — 重新整理與查詢日期
  - `references/08-status-display.md` — 工單狀態說明

## 跨入口共用概念

- **開/清帳**：一般商品與美容入口每日營業前需開帳、結束後清帳，未開帳無法進行銷售
- **會員**：一般商品入口部分功能需要會員；美容商品入口**必須**先設定會員才能新增服務
- **機台模式**：CASH_REGISTER（完整功能）/ PRICE_ONLY（僅查價）/ 工作臺模式（只允許後付款）
- **現金抽屜**：收到現金付款時自動開啟
