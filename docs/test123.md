# Code Review 報告（Staged）

> **時間**：2026/6/2 下午3:13:10
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

新增一個 `numberToAccountingstring` 函式，目的是將數字轉換為**會計格式字串**：負數（或特定條件下的數字）以括號包覆表示（如 `(42)`），其餘則直接轉為字串。屬於單一工具函式的新增，範圍小但邏輯有數個嚴重缺陷。

---

## 2. 潛在問題 🚨

### 2-1. 核心邏輯錯誤（Critical Bug）

```typescript
// ❌ 現行程式碼：判斷 < 9，完全錯誤
if (number < 9) {
  return `(${Math.abs(number)})`;
}
```

會計格式的括號慣例是用來表示**負數**，判斷條件應為 `< 0` 而非 `< 9`。
現行邏輯導致：
- `numberToAccountingstring(5)` → `"(5)"` ✗（正數被誤標為負數）
- `numberToAccountingstring(-3)` → `"(-3)"` → 且 `Math.abs(-3) = 3`，輸出 `"(3)"` ✓（僥倖正確）
- `numberToAccountingstring(10)` → `"10"` ✓
- `numberToAccountingstring(8)` → `"(8)"` ✗（正數 8 被錯誤包覆）

### 2-2. 函式可能回傳 `undefined`

當 `number === null` 或 `number === undefined` 時，函式**沒有顯式 `return`**，會隱性回傳 `undefined`。呼叫端若未防範，會產生執行期錯誤。

```typescript
const result = numberToAccountingstring(null);
result.toUpperCase(); // 💥 TypeError: Cannot read properties of undefined
```

### 2-3. `!=` 非嚴格比較

```typescript
if (number != null) { // 使用非嚴格不等於
```

雖然 `!= null` 在 JS 中可同時篩掉 `null` 與 `undefined`，是一種慣用寫法，但在 TypeScript 專案中應搭配型別守衛（type guard）來明確意圖，避免混淆。

---

## 3. 程式碼品質 📝

### 3-1. 函式命名不符慣例

| 問題 | 說明 |
|---|---|
| `numberToAccountingstring` | `string` 中的 `s` 小寫，應為 `numberToAccountingString`（camelCase） |

### 3-2. 參數型別使用 `any`（TypeScript 反模式）

```typescript
function numberToAccountingstring(number: any) { // ❌
```

使用 `any` 完全放棄了 TypeScript 的型別保護，應改為 `number | null | undefined`，或依實際需求收窄為 `number`。

### 3-3. 無回傳型別標註

函式缺少明確的回傳型別宣告，TypeScript 雖能推斷，但對公開 API 函式而言，明確標註是最佳實踐。

### 3-4. 缺少 JSDoc 文件

工具函式應說明其用途、參數意義與回傳格式。

---

## 4. 效能考量 ⚡

此函式邏輯極為簡單，**無明顯效能問題**。唯一可留意的是：若在高頻渲染場景（如表格每格都呼叫）呼叫 `Math.abs()`，是極低成本的原生運算，無需優化。

---

## 5. 最佳實踐 ✅

| 項目 | 現況 | 建議 |
|---|---|---|
| TypeScript 型別 | `any` | 使用具體型別 |
| 回傳型別 | 缺失 | 加上 `string \| undefined` 或確保總有回傳值 |
| 命名慣例 | `string` 小寫 | `String` 大寫 |
| 邊界處理 | null 時無回傳 | 明確回傳預設值 |
| 模組匯出 | 無 `export` | 視情況加上 `export` |

---

## 6. 改善建議 🔧

### 修正版本（保守修法）

```typescript
/**
 * 將數字轉換為會計格式字串。
 * 負數以括號表示（例：-42 → "(42)"），正數直接轉為字串（例：42 → "42"）。
 * 若傳入 null 或 undefined，回傳空字串。
 *
 * @param value - 要格式化的數字，允許 null 或 undefined
 * @returns 會計格式字串
 */
export function numberToAccountingString(value: number | null | undefined): string {
  if (value == null) {
    return '';
  }

  if (value < 0) {
    return `(${Math.abs(value)})`;
  }

  return value.toString();
}
```

### 進階版本（支援小數位數與千分位）

```typescript
/**
 * 將數字轉換為會計格式字串，支援小數位數控制。
 * 負數以括號表示，正數直接轉為字串。
 *
 * @param value - 要格式化的數字
 * @param fractionDigits - 小數位數（預設 0）
 * @returns 會計格式字串，輸入無效時回傳 ''
 */
export function numberToAccountingString(
  value: number | null | undefined,
  fractionDigits = 0,
): string {
  if (value == null || !isFinite(value)) {
    return '';
  }

  const formatted = Math.abs(value).toFixed(fractionDigits);

  return value < 0 ? `(${formatted})` : formatted;
}

// 使用範例
numberToAccountingString(-1234.5, 2); // "(1234.50)"
numberToAccountingString(1234.5, 2);  // "1234.50"
numberToAccountingString(0);          // "0"
numberToAccountingString(null);       // ""
```

---

## 綜合評分

| 面向 | 評分 |
|---|---|
| 正確性 | ⭐ 2/10 |
| 可讀性 | ⭐⭐⭐ 5/10 |
| TypeScript 合規 | ⭐ 2/10 |
| 完整性 | ⭐⭐ 3/10 |

### 🎯 總評：**2.5 / 10**

> **給分依據**：函式核心邏輯存在嚴重 bug（`< 9` 應為 `< 0`），幾乎所有輸入都會產生錯誤結果；加上 `any` 型別、命名錯誤、無回傳型別、缺乏 null 安全處理，整體品質偏低。
>
> **最值得改進的一點**：**立即修正判斷條件** `number < 9` → `number < 0`，這是影響功能正確性的致命問題，需優先處理。
