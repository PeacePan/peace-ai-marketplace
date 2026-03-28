---
name: member-system-knowledge
description: >
  NorwegianForest 會員系統完整業務知識。涵蓋 posmember、pospet、memberclass、
  pointaccount、givepointsetting、importpoint、pointdiscount、feversocialpoints、
  pointpromotion、petparkcoupon、petparkcouponassign、petparkcouponevent、
  membermerge、memberautoarchive、autoupdatememberclass、cresclabsyncmember，
  以及 Emarsys CRM 同步的 emcontacttask、emfiletask、emwebhook、emsaletask、
  emproducttask、emstoretask 等 24 張表格的欄位定義、業務邏輯、腳本掛勾與表格關聯。
  當需要撰寫或修改任何會員、點數、優惠券、會員等級、Emarsys 同步相關腳本時，務必先讀此 skill。
---

# 會員系統業務知識

NorwegianForest 的會員系統以 `posmember`（會員主檔）為核心，向外延伸出
點數系統、優惠券系統、會員等級管理、寵物管理與各種同步任務。

## 參考文件目錄

### 架構與業務邏輯

| 文件 | 內容摘要 | 何時閱讀 |
|-----|---------|---------|
| [tables-overview.md](references/tables-overview.md) | 24 張表格用途摘要與完整關聯圖 | 第一次了解會員系統架構 |
| [trigger-chains.md](references/trigger-chains.md) | 所有表格間的觸發鏈關係圖（Policy/BeforeInsert/BeforeUpdate/BatchPolicy 觸發路徑） | 排查冗餘觸發、效能問題、或新增跨表寫入前 |
| [posmember-and-pospet.md](references/posmember-and-pospet.md) | 會員主檔與寵物檔的完整欄位、腳本與排程 | 修改會員/寵物基本資料邏輯 |
| [member-class-lifecycle.md](references/member-class-lifecycle.md) | 會員等級設定、等級計算流程、自動更新排程 | 修改會員等級升降邏輯 |
| [point-system.md](references/point-system.md) | 點數帳本、贈點活動、點數折現、點加金、批次匯入、FS 歸戶 | 修改點數相關邏輯 |
| [coupon-system.md](references/coupon-system.md) | 優惠券活動、會員優惠券、指定發放的完整流程 | 修改優惠券相關邏輯 |
| [member-operations.md](references/member-operations.md) | 會員合併、自動封存的流程與注意事項 | 修改會員合併/封存邏輯 |
| [sync-tasks.md](references/sync-tasks.md) | CrescLab LINE 同步、OCard 任務（已棄用） | 修改第三方同步邏輯 |
| [emarsys-sync.md](references/emarsys-sync.md) | Emarsys CRM 同步六張表格（emcontacttask/emfiletask/emwebhook/emsaletask/emproducttask/emstoretask）的推送架構、Webhook 回調、雙向通訊 | 修改 Emarsys 同步相關邏輯 |

### 腳本函式詳細說明

| 文件 | 內容摘要 | 何時閱讀 |
|-----|---------|---------|
| [posmember-scripts-detail.md](references/posmember-scripts-detail.md) | posmember 全部 35 支腳本 + pospet 2 支腳本的函式邏輯說明 | 修改或新增 posmember/pospet 任何腳本前 |
| [point-system-scripts-detail.md](references/point-system-scripts-detail.md) | pointaccount/givepointsetting/importpoint/pointdiscount/feversocialpoints/pointpromotion 共 25 支腳本 | 修改點數相關任何腳本前 |
| [coupon-system-scripts-detail.md](references/coupon-system-scripts-detail.md) | petparkcoupon/petparkcouponevent/petparkcouponassign/memberclass/memberautoarchive/membermerge/autoupdatememberclass/cresclabsyncmember 共 37 支腳本 | 修改優惠券、合併、等級、同步相關腳本前 |
| [emarsys-scripts-detail.md](references/emarsys-scripts-detail.md) | emcontacttask/emfiletask/emwebhook/emsaletask/emproducttask/emstoretask 共 31 支腳本 | 修改任何 Emarsys 同步腳本前 |

## 快速定位

- **會員基本資料問題** → [posmember-and-pospet.md](references/posmember-and-pospet.md)
- **會員等級不正確** → [member-class-lifecycle.md](references/member-class-lifecycle.md) 的等級計算流程
- **點數異動問題** → [point-system.md](references/point-system.md) 的點數帳本邏輯
- **優惠券發放問題** → [coupon-system.md](references/coupon-system.md) 的發放流程
- **表格之間如何關聯** → [tables-overview.md](references/tables-overview.md) 的關聯圖
- **某表格的具體函式邏輯** → 對應的 `*-scripts-detail.md`
- **跨表觸發路徑分析** → [trigger-chains.md](references/trigger-chains.md)
- **操作後為什麼多觸發了幾次 policy** → [trigger-chains.md](references/trigger-chains.md) 的觸發鏈圖
- **會員合併後資料不正確** → [member-operations.md](references/member-operations.md)
- **Emarsys 同步問題（會員資料沒上傳）** → [emarsys-sync.md](references/emarsys-sync.md) 的 emcontacttask
- **Emarsys Webhook 觸發後沒發券/贈點** → [emarsys-sync.md](references/emarsys-sync.md) 的 emwebhook

## 核心業務流程（一句話版）

```
posmember (會員主檔)
  ├── 消費後 → issuePoints → pointaccount (點數記帳)
  ├── 消費後 → issueCoupon → petparkcoupon (優惠券實例)
  ├── 定期排程 → updateMemberClass → memberclass (依消費金額升降等)
  ├── 生日月 → issueBirthMonthCouponAndPoint (發生日點數＋優惠券)
  └── 合併時 → membermerge → (destMember 保留, sourceMember 封存)
```

## 關鍵腳本目錄

```
tables/pos/scripts/posmember/         # 會員主檔腳本（最核心）
tables/pos/scripts/pospet/            # 寵物檔腳本
tables/memberRights/scripts/          # 所有會員權益腳本
  ├── memberclass/                    # 等級計算
  ├── autoupdatememberclass/          # 自動更新等級排程
  ├── pointaccount/                   # 點數帳本
  ├── givepointsetting/               # 贈點活動
  ├── importpoint/                    # 批次匯入點數
  ├── pointdiscount/                  # 點數折現
  ├── pointpromotion/                 # 點加金活動
  ├── feversocialpoints/              # FS 歸戶
  ├── petparkcoupon/                  # 會員優惠券
  ├── petparkcouponassign/            # 優惠券指定發放
  ├── petparkcouponevent/             # 優惠券活動設定
  ├── membermerge/                    # 會員合併
  ├── memberautoarchive/              # 會員自動封存
  └── cresclabsyncmember/            # CrescLab LINE 同步
tables/emarsys/scripts/               # Emarsys CRM 同步腳本
  ├── emcontacttask/                  # 會員資料推送（WebDAV）
  ├── emfiletask/                     # WebDAV 回傳檔案處理
  ├── emwebhook/                      # Webhook 回調處理（發券/贈點）
  ├── emsaletask/                     # 銷售資料推送（SFTP）
  ├── emproducttask/                  # 商品資料推送（SFTP）
  └── emstoretask/                    # 門市資料推送（SFTP）
```
