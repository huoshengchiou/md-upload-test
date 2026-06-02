# Code Review 報告（Staged）

> **時間**：2026/6/2 下午4:30:46
> **工具**：Claude Agent SDK

## 變更統計

```
mock-src/sample-code-a.ts | 9 +++++++++
 1 file changed, 9 insertions(+)
```

---

# Code Review：`mock-src/sample-code-a.ts`

---

## 1. 變更摘要

本次新增一個獨立的 TypeScript 工具函式 `numberToAccountingstring`，其目的是將數字轉換為會計格式字串——負數（或特定條件下的數字）以括號包裹表示，正數則直接轉為字串。

---

## 2. 潛在問題 🐛

### 🔴 嚴重 Bug：判斷條件邏輯錯誤

```typescript
// 原始碼（第 3 行）
if (number < 9) {
```

**會計格式**的核心語意是：**負數**才用括號表示（如 `-100` → `(100)`）。條件應為 `number < 0`，而非 `number < 9`。

目前的邏輯導致：
| 輸入值 | 當前輸出 | 預期輸出 |
|--------|----------|----------|
| `-50` | `(50)` ✅ | `(50)` |
| `0` | `(0)` ❌ | `0` |
| `5` | `(5)` ❌ | `5` |
| `9` | `9` ✅ | `9` |
| `100` | `100` ✅ | `100` |

### 🔴 未處理的隱式 `undefined` 回傳

```typescript
// 第 1–8 行
function numberToAccountingstring(number: any) {
  if (number != null) {  // 若 number 為 null/undefined，函式隱式回傳 undefined
    ...
  }
  // 這裡沒有 return，呼叫端拿到 undefined 可能引發後續錯誤
}
```

呼叫端若直接使用回傳值（如拼接字串），將得到 `"undefined"` 的意外結果。

### 🟡 邊界案例未處理

| 輸入 | 行為 | 問題 |
|------|------|------|
| `NaN` | `NaN != null` 為 `true`，`NaN < 9` 為 `false`，回傳 `"NaN"` | 語意不明確 |
| `Infinity` | `-Infinity < 9` 為 `true`，回傳 `(Infinity)` | 會計格式不應出現無限大 |
| `"5"` (字串) | `"5" < 9` 為 `true`，`Math.abs("5" as any)` 為 `5` | 型別為 `any` 造成靜默錯誤 |

---

## 3. 程式碼品質 📐

### 🟡 函式命名不符合慣例

```typescript
// ❌ 原始：末尾 "string" 的 's' 未大寫
function numberToAccountingstring(...)

// ✅ 建議：完整 camelCase
function numberToAccountingString(...)
```

### 🟡 使用 `any` 型別，喪失 TypeScript 型別保護

```typescript
// ❌ 原始
function numberToAccountingstring(number: any)

// ✅ 建議：明確宣告參數型別與回傳型別
function numberToAccountingString(value: number): string
```

### 🟡 缺少函式說明文件（JSDoc）

公開工具函式應附上用途說明、參數說明與回傳值說明，方便 IDE 提示與團隊維護。

### 🟡 函式未 `export`

若為工具函式，應明確匯出；若為模組私用，應加上註解說明意圖。

---

## 4. 效能考量 ⚡

本函式邏輯簡單，不存在效能瓶頸，但有一點可微調：

```typescript
// 原始（第 7 行）
return number.toString();

// 建議：與第一個 return 風格一致，使用模板字串
return `${number}`;
// 或保留 toString() 亦可，但整體風格應統一
```

---

## 5. 最佳實踐 ✅

### 缺少明確回傳型別標註

TypeScript 強型別的最大優勢在於回傳型別的明確性。現行寫法中，TypeScript 會推斷回傳型別為 `string | undefined`，但這個 `undefined` 是無意為之的邊界缺陷，並非設計。

### 使用 `!= null` 而非嚴格比較

```typescript
// 原始（寬鬆比較，同時排除 null 與 undefined）
if (number != null)

// ✅ TypeScript 中建議改為明確的型別守衛
// 配合型別宣告為 number，可直接移除此防呆，或改寫為：
if (typeof value !== 'number' || !isFinite(value)) {
  throw new TypeError(`Expected a finite number, got: ${value}`);
}
```

---

## 6. 改善建議 🔧

以下為完整重構範例：

```typescript
/**
 * 將數字轉換為會計格式字串。
 * - 負數以括號表示，例如 -100 → "(100)"
 * - 零與正數直接轉為字串，例如 0 → "0"、100 → "100"
 *
 * @param value - 要轉換的有限數字
 * @returns 會計格式字串
 * @throws {TypeError} 若輸入值不是有限數字
 *
 * @example
 * numberToAccountingString(-50)  // "(50)"
 * numberToAccountingString(0)    // "0"
 * numberToAccountingString(100)  // "100"
 */
export function numberToAccountingString(value: number): string {
  if (typeof value !== 'number' || !isFinite(value)) {
    throw new TypeError(`Expected a finite number, got: ${value}`);
  }

  if (value < 0) {
    return `(${Math.abs(value)})`;
  }

  return `${value}`;
}
```

**重構重點說明：**
1. ✅ 修正條件 `< 9` → `< 0`（核心 Bug 修正）
2. ✅ 明確型別：`number` 取代 `any`，加上回傳型別 `: string`
3. ✅ 防禦性型別守衛：拒絕 `NaN`、`Infinity` 等非法輸入並拋出明確錯誤
4. ✅ 加上 `export` 與 JSDoc 文件
5. ✅ 修正函式名稱 `numberToAccountingString`
6. ✅ 統一使用模板字串風格

---

## 綜合評分

> **3 / 10**

### 給分依據

| 面向 | 狀況 |
|------|------|
| 邏輯正確性 | ❌ 核心條件 `< 9` 是錯誤的，會計格式應為 `< 0` |
| 型別安全 | ❌ 使用 `any`，放棄 TypeScript 最大優勢 |
| 邊界處理 | ❌ `null`/`undefined` 時隱式回傳 `undefined`，NaN/Infinity 未處理 |
| 命名規範 | ⚠️ `numberToAccountingstring` 命名有誤 |
| 可維護性 | ⚠️ 無文件、無匯出 |
| 結構簡潔 | ✅ 函式本體短小，意圖基本清楚 |

### 最值得改進的一點

**立即修正第 3 行的邏輯錯誤** `number < 9` → `number < 0`。這是影響所有呼叫端正確性的核心 Bug，其餘改善屬於品質提升，但此項是功能正確性的根本問題。
