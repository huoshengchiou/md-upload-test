# Code Review 報告（Staged）

> **時間**：2026/6/2 下午3:48:36
> **工具**：Claude Agent SDK

## 變更統計

```
mock-src/sample-code-a.ts | 9 +++++++++
 1 file changed, 9 insertions(+)
```

---

# Code Review 報告

## 1. 變更摘要

本次新增一個 `numberToAccountingstring` 函式，位於 `mock-src/sample-code-a.ts`。  
功能意圖為將數字轉換為**會計格式字串**（accounting format），即負數以括號表示，例如 `-100` → `(100)`。  
變更範圍小（9 行），但存在一個**嚴重邏輯錯誤**及多項型別與品質問題。

---

## 2. 潛在問題

### 🔴 嚴重 Bug：閾值條件錯誤

```typescript
// 現有程式碼（第 3 行）
if (number < 9) {
```

這裡的 `9` 應為 `0`。**會計格式的規則是：負數才需要加括號**，而非「小於 9 的數」。

| 輸入 | 現有行為 | 預期行為 |
|------|----------|----------|
| `5`  | `"(5)"` ❌ | `"5"` |
| `8`  | `"(8)"` ❌ | `"8"` |
| `-5` | `"(5)"` ✅ | `"(5)"` |
| `10` | `"10"` ✅ | `"10"` |

---

### 🟡 邊界案例未處理

| 輸入 | 現有行為 | 風險 |
|------|----------|------|
| `NaN` | `"NaN"` | 可能造成顯示異常 |
| `Infinity` / `-Infinity` | `"(Infinity)"` / `"Infinity"` | 不合理的輸出 |
| `0` | `"(0)"` ❌ | 0 不應加括號 |
| `null` / `undefined` | `undefined`（無回傳值） | 呼叫端若未處理會有 runtime 錯誤 |

---

### 🟡 函式可能回傳 `undefined`

當傳入 `null` 時，函式沒有任何 `return`，隱式回傳 `undefined`，但型別宣告上完全看不出這一點，容易誤用。

---

## 3. 程式碼品質

### 🔴 命名問題：違反 camelCase 規範

```typescript
// ❌ 現有
function numberToAccountingstring(...)

// ✅ 應為
function numberToAccountingString(...)
```

TypeScript / JavaScript 命名慣例中，複合詞每個單字首字母均應大寫（`String` 而非 `string`）。

---

### 🔴 參數型別使用 `any`

```typescript
// ❌ 現有
function numberToAccountingstring(number: any)

// ✅ 應為
function numberToAccountingString(value: number): string
```

使用 `any` 完全失去 TypeScript 靜態型別的優勢：
- 無法在編譯期攔截錯誤（例如傳入字串 `"abc"`）
- IDE 無法提供準確的自動補全與型別推斷

---

### 🟡 參數命名遮蔽全域 `number` 型別

```typescript
function numberToAccountingstring(number: any) { ... }
//                                 ^^^^^^ 這會遮蔽內建型別 `number`
```

建議改用 `value` 或 `amount` 作為參數名稱，避免語意混淆。

---

### 🟡 缺少回傳型別宣告與 JSDoc

明確宣告回傳型別可提升可讀性與型別安全：

```typescript
function numberToAccountingString(value: number): string { ... }
```

---

## 4. 效能考量

本函式邏輯簡單，**無明顯效能問題**。  
唯一需注意的是 `Math.abs()` 在傳入非數字時（因使用 `any`）行為不確定，但在正確型別限制下不是問題。

---

## 5. 最佳實踐

### 應符合的 TypeScript 慣例

- ✅ 明確宣告參數與回傳型別（避免 `any`）
- ✅ 使用 `=== null` 或 TypeScript optional 機制取代 `!= null`（雖然此處 `!= null` 同時攔截 `undefined`，是合法慣用法，但加上型別限制後此判斷可完全移除）
- ✅ 函式名稱遵循 camelCase

---

## 6. 改善建議

### 完整重構範例

```typescript
/**
 * 將數字轉換為會計格式字串。
 * - 負數以括號表示，例如：-100 → "(100)"
 * - 零與正數直接轉為字串，例如：100 → "100"
 *
 * @param value - 要格式化的數字
 * @returns 會計格式字串
 */
function numberToAccountingString(value: number): string {
  if (!isFinite(value) || isNaN(value)) {
    throw new RangeError(`無效的數字: ${value}`);
  }

  if (value < 0) {
    return `(${Math.abs(value)})`;
  }

  return value.toString();
}
```

### 若需支援 `null` / `undefined` 的可選版本

```typescript
function numberToAccountingString(value: number | null | undefined): string | null {
  if (value == null) return null;

  if (!isFinite(value) || isNaN(value)) {
    throw new RangeError(`無效的數字: ${value}`);
  }

  return value < 0 ? `(${Math.abs(value)})` : value.toString();
}
```

---

## 綜合評分

> **評分：2 / 10**

### 給分依據

| 面向 | 狀態 |
|------|------|
| 函式結構清晰，可讀性尚可 | ✅ |
| 存在嚴重邏輯錯誤（`< 9` 應為 `< 0`） | ❌ |
| 使用 `any` 型別 | ❌ |
| 命名不符慣例 | ❌ |
| 邊界案例未處理（`NaN`、`Infinity`、`0`） | ❌ |
| 無回傳型別宣告 | ❌ |

### 最值得改進的一點

**立即修正第 3 行的閾值**：將 `number < 9` 改為 `number < 0`。  
這是一個會導致**所有正數 0–8 被錯誤格式化**的功能性 Bug，在上線前必須修復，其餘品質問題可於後續重構中逐步改善。
