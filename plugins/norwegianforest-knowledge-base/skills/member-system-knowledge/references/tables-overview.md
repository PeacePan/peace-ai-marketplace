# 會員系統表格總覽

> **版本號說明**：各表格標注的版本號對應程式碼中 `version: isDev ? null : N` 的 N 值。
> 若知識文件記錄的版本低於當前程式碼版本，代表該表格可能已有更新，需重新確認腳本邏輯。

## 24 張表格用途

### 核心會員檔（TABLE_GROUP.POS）

| 表格 | 版本 | 顯示名稱 | 用途 |
|-----|------|---------|-----|
| `posmember` | v344 | 會員檔 | 整個會員系統的核心主檔，記錄基本資料、等級、點數狀態、Emarsys 同步欄位 |
| `pospet` | v4 | 寵物檔 | 紀錄會員飼養的寵物資料，關聯 posmember |

### 會員等級管理（TABLE_GROUP.MEMBER_RIGHTS）

| 表格 | 版本 | 顯示名稱 | 用途 |
|-----|------|---------|-----|
| `memberclass` | v3 | 會員等級設定檔 | 設定升級金卡/黑卡所需消費金額，以及續等條件 |
| `autoupdatememberclass` | v9 | 自動更新會員等級排程檔 | 補更新特定日期的會員等級，適用於補跑歷史等級計算 |

### 會員操作

| 表格 | 版本 | 顯示名稱 | 用途 |
|-----|------|---------|-----|
| `membermerge` | v21 | 會員合併檔 | 合併兩個會員帳號（保留 dest，封存 source） |
| `memberautoarchive` | v3 | 會員自動封存檔 | 新增紀錄後直接觸發封存指定會員 |

### 點數系統

| 表格 | 版本 | 顯示名稱 | 用途 |
|-----|------|---------|-----|
| `pointaccount` | v50 | 點數帳本檔 | 每次點數異動都新增一筆，記錄餘額與來源 |
| `givepointsetting` | v32 | 贈點活動設定檔 | 設定贈送點數的規則（消費滿額、倍數、特定渠道等） |
| `importpoint` | v29 | 匯入點數檔 | 批次指定會員名單，按設定發送點數 |
| `pointdiscount` | v11 | 點數折現活動設定檔 | 設定使用點數折抵訂單金額的比例 |
| `feversocialpoints` | v5 | FS 點數歸戶檔 | 接收 FeverSocial 點數歸戶資訊，自動發點給會員 |
| `pointpromotion` | v15 | 點數消費活動設定檔 | 設定點加金活動，指定兌換商品與所需點數 |

### 優惠券系統

| 表格 | 版本 | 顯示名稱 | 用途 |
|-----|------|---------|-----|
| `petparkcouponevent` | v36 | 優惠券活動檔 | 定義優惠券活動規則（類型、發放條件、使用條件） |
| `petparkcoupon` | v92 | 會員優惠券檔 | 發放給特定會員的優惠券實例，記錄使用狀態 |
| `petparkcouponassign` | v25 | 優惠券指定發放檔 | 批次指定會員名單發放優惠券 |

### 同步任務（第三方平台）

| 表格 | 版本 | 顯示名稱 | 用途 |
|-----|------|---------|-----|
| `cresclabsyncmember` | v15 | CrescLab 同步會員 | 透過漸強 API 同步會員的 LINE 訂閱狀態 |
| `ocardsyncpointtask` | v4 | OCard 會員同步點數任務檔 | **@deprecated** 點數移轉完畢，保留作歷史紀錄 |
| `ocardsynctask` | v12 | OCard 會員同步任務檔 | **@deprecated** 會員資料移轉完畢，保留作歷史紀錄 |

### Emarsys CRM 同步（TABLE_GROUP.EMARSYS）

| 表格 | 版本 | 顯示名稱 | 用途 |
|-----|------|---------|-----|
| `emcontacttask` | v49 | Emarsys 會員任務檔 | 將 posmember 批次打包成 CSV 上傳至 Emarsys WebDAV |
| `emfiletask` | v2 | Emarsys 發放排程檔 | 掃描 WebDAV 回傳檔案，執行點數/優惠券發放 |
| `emwebhook` | v28 | Emarsys Webhook 紀錄表 | 接收 Emarsys Webhook 回調，觸發發券/贈點 |
| `emsaletask` | v46 | Emarsys 銷售資料同步檔 | 將銷售/退貨資料批次推送至 Emarsys SFTP |
| `emproducttask` | v19 | Emarsys 商品任務檔 | 將料件/服務料件資料每日推送至 Emarsys SFTP |
| `emstoretask` | v11 | Emarsys 門市任務檔 | 將門市資料每小時推送至 Emarsys SFTP |

---

## 表格關聯圖

```
posmember（會員主檔） ← 整個會員系統的核心中心
    │
    ├── pospet（寵物檔）
    │     └── memberName → posmember.name
    │
    ├── pointaccount（點數帳本）
    │     └── memberName → posmember.name
    │     └── 來源可為：possale, posreturn, posservicesale, posservicereturn,
    │                   importpoint, mashoporder, mashoprefund, posmember,
    │                   mapartialorder, mapartialrefund, feversocialpoints
    │
    ├── feversocialpoints（FS 歸戶）
    │     └── memberName → posmember.name
    │     └── 新增後 → 觸發 pointaccount（新增點數）
    │
    ├── petparkcoupon（會員優惠券）
    │     └── memberName → posmember.name
    │     └── couponEventName → petparkcouponevent.name
    │     └── 來源可為：possale, posservicesale, mashoporder, petparkcouponassign
    │
    ├── importpoint（批次匯入點數）
    │     └── memberList[].memberName → posmember.name
    │     └── 執行後 → 觸發 pointaccount（新增點數）
    │
    ├── petparkcouponassign（優惠券指定發放）
    │     └── assignList[].memberName → posmember.name
    │     └── couponName → petparkcouponevent.name
    │     └── 執行後 → 觸發 petparkcoupon（新增優惠券實例）
    │
    ├── membermerge（會員合併）
    │     └── destMemberName → posmember.name （保留會員）
    │     └── sourceMemberName → posmember.name （封存會員）
    │
    └── memberautoarchive（自動封存）
          └── memberName → posmember.name
          └── 新增後 → 直接封存該 posmember

petparkcouponevent（優惠券活動）
    └── 被 petparkcoupon.couponEventName 參照
    └── 被 petparkcouponassign.couponName 參照

memberclass（等級設定）
    └── 由 posmember.updateMemberClass 函式讀取計算升降等條件

autoupdatememberclass（自動更新等級排程）
    └── 排程觸發後 → 呼叫 posmember.補更新會員等級 function

cresclabsyncmember（CrescLab 同步）
    └── 排程觸發後 → 更新 posmember.emLineIdStatus

givepointsetting（贈點活動）
    └── 被 posmember.issuePoints function 讀取規則
    └── 被 pointaccount 紀錄 sourceTable = possale/posservicesale 時使用

pointdiscount（點數折現）
    └── 被 POS 結帳流程讀取折現設定

pointpromotion（點加金活動）
    └── 被 POS 結帳流程讀取兌換設定
    └── redeemItems[].itemName → item.name
```

---

## 關鍵外鍵對應關係

| 欄位 | 所在表格 | 對應表格 | 說明 |
|-----|---------|---------|-----|
| `memberName` | 多張表格 | posmember.name | 關聯會員主檔 |
| `memberName` | pospet | posmember.name | 寵物歸屬的會員 |
| `memberName` | pointaccount | posmember.name | 點數帳本的會員 |
| `couponEventName` | petparkcoupon | petparkcouponevent.name | 優惠券的活動來源 |
| `couponName` | petparkcouponassign | petparkcouponevent.name | 發放批次對應的活動 |
| `destMemberName` | membermerge | posmember.name | 合併後保留的會員 |
| `sourceMemberName` | membermerge | posmember.name | 合併後封存的會員 |

---

## 各表格腳本掛勾摘要

| 表格 | 掛勾類型 | 說明 |
|-----|---------|-----|
| `posmember` | POLICY + BEFORE_UPDATE | 驗證欄位、觸發消費資料更新 |
| `pospet` | POLICY + BEFORE_INSERT | 驗證寵物資料、新增前處理 |
| `petparkcoupon` | BATCH_POLICY + BEFORE_INSERT | 批次發券驗證、新增前產生券號 |
| `petparkcouponevent` | BEFORE_UPDATE + 多個 POLICY | 更新前驗證、門市群組/料件篩選器驗證 |
| `givepointsetting` | BEFORE_UPDATE + POLICY | 更新前處理、驗證贈點設定 |
| `memberautoarchive` | POLICY | 新增後直接觸發封存 |
| `membermerge` | POLICY | 新增後觸發合併排程 |
| `autoupdatememberclass` | POLICY | 驗證排程操作權限 |
| `importpoint` | POLICY | 驗證匯入設定 |
| `petparkcouponassign` | POLICY | 驗證發放設定 |
| `pointaccount` | POLICY | 驗證點數異動操作 |
| `pointdiscount` | POLICY | 驗證折現設定 |
| `pointpromotion` | POLICY | 驗證點加金設定 |
| `cresclabsyncmember` | POLICY | 驗證漸強同步任務 |

---

## 各表格排程摘要

| 表格 | 排程名稱 | 頻率 | 說明 |
|-----|---------|-----|-----|
| `posmember` | 更新會員優惠券排程 | HOURLY | 更新會員的優惠券欄位 |
| `posmember` | 發放消費贈送優惠券排程 | FIVE_MINS | 當天消費的會員發券 |
| `posmember` | 會員與寵物生日月發點發券排程 | FIVE_MINS | 每月1日發生日優惠 |
| `posmember` | 發放會員等級異動優惠券排程 | HOURLY | 補發等級異動優惠券 |
| `posmember` | 發放會員等級異動點數排程 | HOURLY | 補發等級異動點數 |
| `posmember` | 更新會員點數排程 | FIVE_MINS | 更新點數相關欄位 |
| `posmember` | 贈送點數執行排程 | FIVE_MINS | 執行待處理的贈點 |
| `posmember` | 更新會員等級排程 | FIVE_MINS | 依消費金額計算等級 |
| `posmember` | 每日更新會員銷售資料排程 | DAILY (07:00 台灣) | 更新消費統計欄位 |
| `pointaccount` | 過期點數處理排程 | FIVE_MINS (prod) | 扣除已過期的點數 |
| `pointaccount` | 已派發點數批次封存 | FIVE_MINS (prod only) | 封存已派發的點數 |
| `membermerge` | 會員合併排程 | FIVE_MINS | 執行待合併的任務 |
| `importpoint` | 匯入點數排程 | HOURLY | 執行待匯入的點數任務 |
| `petparkcouponassign` | 優惠券發放排程 | FIVE_MINS | 執行待發放的優惠券任務 |
| `autoupdatememberclass` | 更新日期排程 | TEN_MINS | 補跑等級更新排程 |
| `petparkcoupon` | 類型補值排程 | TEN_MINS | 補填優惠券類型欄位 |
| `pointpromotion` | 更新已兌換數量 | HALF_HOURLY | 計算點加金兌換數量 |
| `cresclabsyncmember` | 建立漸強存取任務 | HOURLY (prod) | 建立 LINE 狀態同步任務 |
| `cresclabsyncmember` | 取得漸強會員資料 | HOURLY (prod) | 下載漸強 LINE 資料 |
| `cresclabsyncmember` | LINE 同步會員狀態 | HOURLY (prod) | 同步至 posmember |
