# 進階 Google 專利搜尋指南：鄰近與邏輯

掌握 Google 專利需要超越簡單的關鍵字搜尋。本指南解釋如何使用鄰近運算子與結構化的布林邏輯，來找出基本搜尋會錯過的「隱藏」專利。

## 1. 布林基礎（基礎）

所有運算子必須全部大寫。

- **AND**：兩個詞彙必須同時出現在文件中任意位置。
- **OR**：任一詞彙出現（用於同義詞：GaN OR "Gallium Nitride"）。
- **NOT**：排除詞彙（GaN NOT "LED"）。

## 2. 鄰近運算子（精準工具）

鄰近運算子是專利研究者武器庫中最強大的工具。它們定義兩個詞彙必須彼此多近。

### A. ADJ/x（相鄰）

ADJ 運算子要求詞彙以您輸入的確切順序出現，中間分隔不超過 x 個詞彙。

- **為什麼使用它？** 找出連結的概念，但中間可能有描述性詞彙。
- **範例**：wireless ADJ/5 charging
  - **匹配**："Wireless charging," "Wireless inductive charging," "Wireless high-speed resonance charging."
  - **不匹配**："Charging the device through a wireless connection"（錯誤順序）。
- **專業提示**：如果您希望兩個詞彙緊鄰，但想避免某些資料庫變體中引號的嚴格性，請使用 ADJ/1。

### B. NEAR/x（附近）

類似 ADJ，但順序不重要。

- **為什麼使用它？** 當兩個概念在句子中相關，但您不知道作者先提到哪一個。
- **範例**：sensor NEAR/10 "autonomous vehicle"
  - **匹配**："...an autonomous vehicle equipped with a LIDAR sensor..."
  - **匹配**："...the sensor transmits data to the autonomous vehicle..."

### C. SAME（相同段落）

SAME 運算子是「廣泛網」鄰近工具。它尋找在相同段落或彼此約 200 個詞彙內的詞彙。

- **為什麼使用它？** 如果兩個詞彙在相同段落中，它們幾乎一定是相同技術討論的一部分。如果它們在專利的不同部分中，它們可能無關。
- **範例**：semiconductor SAME "heat sink"
  - **匹配**：描述半導體如何安裝到散熱器的段落。
  - **過濾掉**：在背景中提到「semiconductor」，而在結尾的通用零件清單中僅提到「heat sink」的專利。

## 3. 進階巢狀（運算順序）

就像代數一樣，使用括號 ( ) 告訴 Google 要先執行搜尋的哪一部分。

專業搜尋的「標準」公式：  
(同義詞群組 1) AND (同義詞群組 2) NOT (排除群組)

**範例**（高品質 GaN 搜尋）：  
("Gallium Nitride" OR GaN) AND (semiconductor ADJ/5 crystal) NOT ("wide bandgap" OR LED)

## 4. 欄位範圍（搜尋特定部分）

您可以將搜尋限制在專利的特定部分，以增加相關性。

| 前綴 | 目標部分 | 最佳用途...                  |
| ---- | -------- | ---------------------------- |
| TI=  | 標題     | 高相關性的廣泛命中。         |
| AB=  | 摘要     | 找出發明的「要旨」。         |
| CL=  | 權利要求 | 最重要的部分；定義法律保護。 |
| IN=  | 發明人   | 找出領域中的特定專家。       |
| PA=  | 受讓人   | 找出特定公司擁有的專利。     |

## 5. 實務比較：我該用哪一個？

| 如果您想找出...  | 使用這個... | 範例                           |
| ---------------- | ----------- | ------------------------------ |
| 確切的技術術語   | 引號        | "Phase change memory"          |
| 元件的特徵       | ADJ/x       | lens ADJ/3 zoom                |
| 兩個元件共同運作 | NEAR/x      | battery NEAR/15 "cooling loop" |
| 廣泛的技術關係   | SAME        | graphene SAME conductivity     |

## 6. 常見陷阱避免

1. **過度 AND**：對同義詞使用 AND（例如，GaN AND "Gallium Nitride"）。這強制專利必須包含兩個，往往排除 50% 的相關結果。對同義詞使用 OR。
2. **缺少萬用字元**：使用 _ 捕捉複數或變體。Sensing ADJ/5 gas 可能錯過 Sensors ADJ/5 gases。使用 sens_ ADJ/5 gas\*。
3. **大小寫敏感**：Google 專利通常對關鍵字不區分大小寫，但布林運算子必須大寫。

---

# Advanced Google Patents Search Guide: Proximity & Logic

Mastering Google Patents requires moving beyond simple keyword searches. This guide explains how to use proximity operators and structured Boolean logic to find "hidden" patents that basic searches miss.

## 1. Boolean Basics (The Foundation)

All operators must be in ALL CAPS.

- **AND**: Both terms must appear anywhere in the document.
- **OR**: Either term appears (use for synonyms: GaN OR "Gallium Nitride").
- **NOT**: Excludes a term (GaN NOT "LED").

## 2. Proximity Operators (The Precision Tools)

Proximity operators are the most powerful tools in a patent researcher's arsenal. They define how close two words must be to one another.

### A. ADJ/x (Adjacent)

The ADJ operator requires words to appear in the exact order you typed them, separated by no more than x words.

- **Why use it?** To find concepts that are linked but might have descriptive words in between.
- **Example**: wireless ADJ/5 charging
  - **Matches**: "Wireless charging," "Wireless inductive charging," "Wireless high-speed resonance charging."
  - **Does NOT match**: "Charging the device through a wireless connection" (wrong order).
- **Pro Tip**: Use ADJ/1 if you want two words right next to each other but want to avoid the strictness of quotes in some database variations.

### B. NEAR/x (Near)

Similar to ADJ, but the order does not matter.

- **Why use it?** When two concepts are related in a sentence, but you don't know which one the author mentioned first.
- **Example**: sensor NEAR/10 "autonomous vehicle"
  - **Matches**: "...an autonomous vehicle equipped with a LIDAR sensor..."
  - **Matches**: "...the sensor transmits data to the autonomous vehicle..."

### C. SAME (Same Paragraph)

The SAME operator is a "wide net" proximity tool. It looks for words within the same paragraph or within approximately 200 words of each other.

- **Why use it?** If two words are in the same paragraph, they are almost certainly part of the same technical discussion. If they are in different sections of the patent, they might be unrelated.
- **Example**: semiconductor SAME "heat sink"
  - **Matches**: A paragraph describing how a semiconductor is mounted to a heat sink.
  - **Filters out**: A patent that mentions "semiconductor" in the Background and "heat sink" only in a list of generic parts at the end.

## 3. Advanced Nesting (Order of Operations)

Just like in algebra, use parentheses ( ) to tell Google which part of the search to execute first.

The "Standard" Formula for Professional Searching:  
(Synonym Group 1) AND (Synonym Group 2) NOT (Exclusion Group)

**Example** (A high-quality GaN search):  
("Gallium Nitride" OR GaN) AND (semiconductor ADJ/5 crystal) NOT ("wide bandgap" OR LED)

## 4. Field Scoping (Searching Specific Sections)

You can restrict your search to specific parts of the patent to increase relevancy.

| Prefix | Targeted Section | Best For...                                               |
| ------ | ---------------- | --------------------------------------------------------- |
| TI=    | Title            | High-relevancy broad hits.                                |
| AB=    | Abstract         | Finding the "gist" of the invention.                      |
| CL=    | Claims           | The most important section; defines the legal protection. |
| IN=    | Inventor         | Finding specific experts in the field.                    |
| PA=    | Assignee         | Finding patents owned by a specific company.              |

## 5. Practical Comparison: Which one do I use?

| If you want to find...          | Use this... | Example                        |
| ------------------------------- | ----------- | ------------------------------ |
| An exact technical term         | Quotes      | "Phase change memory"          |
| A feature of a component        | ADJ/x       | lens ADJ/3 zoom                |
| Two components working together | NEAR/x      | battery NEAR/15 "cooling loop" |
| A broad technical relationship  | SAME        | graphene SAME conductivity     |

## 6. Common Pitfalls to Avoid

1. **Over-ANDing**: Using AND for synonyms (e.g., GaN AND "Gallium Nitride"). This forces the patent to contain both, which often excludes 50% of relevant results. Use OR for synonyms.
2. **Missing Wildcards**: Use _ to catch plurals or variations. Sensing ADJ/5 gas might miss Sensors ADJ/5 gases. Use sens_ ADJ/5 gas\*.
3. **Case Sensitivity**: Google Patents is usually case-insensitive for keywords, but Boolean operators MUST be capitalized.
