# 會員主檔（posmember）與寵物檔（pospet）

## posmember 會員主檔

**TABLE_NAME**: `POS_MEMBER`
**檔案路徑**: `tables/pos/posmember.ts`
**腳本路徑**: `tables/pos/scripts/posmember/`
**JobQueues**: 20 個 queue，各 10 並發（高吞吐量表格）

### 欄位群組

| 群組 | 主要欄位 |
|-----|---------|
| 基本資料（BASIC） | name（會員編號）、source（來源）、kgMemo、policy、pointArchivedAt |
| 個人資訊（INFO） | displayName、mobile、email、gender、birth、address、invoiceCarrier、eGUIUniNo |
| Emarsys 同步（EMARSYS） | emLineId、emLineIdStatus、birthMonth、registeredAt、memberClass、memberStartAt、memberEndAt、consumerlifeTotal、consumerlifeCount、各等級升降差額、點數/優惠券狀態欄位、寵物欄位（01~03）... |
| 系統（SYSTEM） | refreshConsumptionAt、refreshConsumptionError |

### 關鍵計算欄位（scriptReadWrite: UPDATE，由系統計算）

| 欄位 | 說明 |
|-----|-----|
| `memberClass` | 會員等級：一般/金卡/黑卡 |
| `memberStartAt` | 會籍起日 |
| `memberEndAt` | 會籍迄日 |
| `consumerlifeTotal` | 會籍期間消費總金額（升降等依據） |
| `consumerlifeCount` | 會籍期間消費次數 |
| `goldLevelupSaleTotal` | 距離升金卡差額 |
| `blackLevelupSaleTotal` | 距離升黑卡差額 |
| `goldHoldSaleTotal` | 金卡續等差額 |
| `blackHoldSaleTotal` | 黑卡續等差額 |
| `pointsValid` | 有效消費點數 |
| `pointsEndTime` | 最近點數效期迄日 |
| `pointsStatus` | 點數狀態（待更新/錯誤/完成） |
| `couponCnt` | 有效優惠券張數 |
| `couponStatus` | 優惠券狀態（待更新/錯誤/完成） |
| `issueBirthMonthPointAt` | 發放生日點數日期 |
| `issueBirthMonthCouponAt` | 發放生日優惠券日期 |
| `lastConsumedAt` | 上一次消費日期 |
| `emSyncStatus` | Emarsys 同步狀態（待同步/已完成） |

### 表身（lines）

| 表身 | 說明 |
|-----|-----|
| `tags` | 標籤 |
| `categories` | 飼養寵物分類 |
| `memos` | 內部備註（含門市、使用者） |
| `membershipLog` | 會籍歷程（等級、起迄日、優惠券/點數發放狀態） |
| `categoryTags` | 品類標籤（銷售後更新） |

### 腳本掛勾

| 掛勾 | 檔案 | 說明 |
|-----|-----|-----|
| POLICY | `scripts/posmember/policy.ts` | 主要驗證邏輯 |
| BEFORE_UPDATE | `scripts/posmember/beforeUpdate.ts` | 更新前觸發（消費資料更新等） |

### 核心函式

#### 會員權益 - 優惠券
| 函式 | 檔案 | 說明 |
|-----|-----|-----|
| 更新會員優惠券 | `updateCoupons.ts` | 計算並更新優惠券欄位 |
| 發放消費贈送優惠券 | `issueConsumptionCoupon.ts` | 消費後發放優惠券 |
| 會員與寵物生日月發點發券 | `issueBirthMonthCouponAndPoint.ts` | 生日月發放優惠券+點數 |
| 發放會員等級異動優惠券 | `issueClassChangeCoupon.ts` | 等級異動時發放優惠券 |

#### 會員權益 - 點數
| 函式 | 檔案 | 說明 |
|-----|-----|-----|
| 更新會員點數 | `updatePoints.ts` | 計算並更新點數欄位 |
| 贈送點數 | `issuePoints.ts` | 依贈點活動對消費進行贈點 |
| 重算會員剩餘點數 | `recalcPointBalance.ts` | 重新計算所有點數帳本餘額 |
| 發放會員等級異動點數 | `issueClassChangePoints.ts` | 等級異動時發放點數 |

#### 會員權益 - 會員等級
| 函式 | 檔案 | 說明 |
|-----|-----|-----|
| 更新會員等級 | `updateMemberClass.ts` | 依消費金額計算並更新等級 |
| 補更新會員等級 | `patchUpdateMemberClass.ts` | 補跑歷史等級計算 |

#### Emarsys 同步
| 函式 | 檔案 | 說明 |
|-----|-----|-----|
| 更新會員銷售資料 | `refreshConsumption.ts` | 抓取消費資料、計算統計欄位 |
| 完成結帳後更新會員資料 | `emUpdateMemberAfterFinishCheckout.ts` | 結帳完成後觸發 |
| 更新會員品類標籤 | `emUpdateMemberCategoryTag.ts` | 更新品類標籤狀態 |

#### Magento 串接
| 函式 | 檔案 | 說明 |
|-----|-----|-----|
| 建立會員 | `maCreateMember.ts` | Magento 呼叫建立會員 |
| 更新會員 | `maUpdateMember.ts` | Magento 呼叫更新會員 |

### 排程列表

| 排程 | 頻率 | 說明 |
|-----|-----|-----|
| 更新會員優惠券排程 | HOURLY | 定期更新優惠券欄位 |
| 發放消費贈送優惠券排程 | FIVE_MINS | 每5分鐘處理待發優惠券 |
| 會員與寵物生日月發點發券排程 | FIVE_MINS | 每月1日觸發生日優惠 |
| 發放會員等級異動優惠券排程 | HOURLY | 補發因錯誤漏發的等級異動券 |
| 發放會員等級異動點數排程 | HOURLY | 補發因錯誤漏發的等級異動點數 |
| 更新會員點數排程 | FIVE_MINS | 定期更新點數欄位 |
| 贈送點數執行排程 | FIVE_MINS | 執行待贈點任務 |
| 更新會員等級排程 | FIVE_MINS | 依最新消費計算等級 |
| 重跑所有會員的更新會員等級排程 | FIVE_MINS | 全量等級重算 |
| 更新會員分批取排程 | HOURLY | 更新分批取商品欄位 |
| 檢查會員分批取過期排程 | HALF_HOURLY | 檢查已過期的分批取 |
| 每日更新會員銷售資料排程 | DAILY 07:00 台灣 | 批次計算消費統計 |

### 注意事項

1. **emSyncStatus 觸發時機**：
   - 產生已付款的銷售單/服務銷售單或已完成的銷退單/服務銷退單
   - emSync 為是且 Emarsys 相關欄位變動
   - 產生分批取訂單/兌換單/退貨單/退款單

2. **memberClass 更新邏輯**：
   - 依 memberclass 表設定的消費金額計算
   - consumerlifeTotal 為會籍期間消費總金額
   - consumerlifeCount 為消費次數（金卡/黑卡有最低消費次數要求）

3. **pointArchivedAt**：
   - 會員合併後，被封存的會員的點數帳本封存時間
   - 封存後該會員的點數不再計算

---

## pospet 寵物檔

**TABLE_NAME**: `POS_PET`
**檔案路徑**: `tables/pos/pospet.ts`
**腳本路徑**: `tables/pos/scripts/pospet/`

### 主要欄位

| 欄位 | 類型 | 說明 |
|-----|-----|-----|
| `name` | KEY（自動編碼） | 寵物編號，8位數字 |
| `displayName` | STRING | 寵物名稱 |
| `species` | REF → pos_pet_species | 品種，關聯品種表 |
| `category` | ENUM | 分類（狗/貓/其他） |
| `status` | ENUM（可為空） | 狀態 |
| `gender` | ENUM（可為空） | 性別 |
| `startDate` | DATE（可為空） | 生日 |
| `memberName` | REF → posmember（可為空） | 歸屬會員，支援腳本更新（會員合併時更新） |
| `userName` | REF → user（可為空） | 歸屬使用者 |
| `memo` | STRING（可為空） | 備註 |
| `monthCardMemo` | STRING（可為空） | 紙本月卡編號 |

### 表身

| 表身 | 說明 |
|-----|-----|
| `tags` | 標籤 |

### 腳本掛勾

| 掛勾 | 檔案 | 說明 |
|-----|-----|-----|
| POLICY | `scripts/pospet/policy.ts` | 驗證寵物資料 |
| BEFORE_INSERT | `scripts/pospet/beforeInsert.ts` | 新增前處理 |

### 注意事項

- `memberName` 欄位雖設為 `INSERT_NULLABLE_REF_KEY_FIELD` 但 `scriptReadWrite: UPDATE`，
  主要是為了**會員合併時**可以批次更新寵物的歸屬會員
- 品種資料存於另一張 `pos_pet_species` 表，含 `category`、`memo` 等欄位
