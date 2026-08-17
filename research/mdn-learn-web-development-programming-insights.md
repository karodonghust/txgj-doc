# MDN《Learn Web Development》詳盡審閱：程序設計洞見與工程注意事項

> 研究日期：2026-07-30（Asia/Hong_Kong）  
> 審閱入口：[MDN 學習 Web 開發](https://developer.mozilla.org/zh-CN/docs/Learn_web_development)  
> 來源原則：只採用 MDN、WHATWG、W3C、ECMAScript、IETF 等規範制定者或技術維護者的一手資料；以 MDN 課程原文為主。  
> 定位：這不是課程目錄摘要，而是把整套課程轉譯成可用於真實專案的設計規則、不變量、失敗模式和檢查表。

## 一、結論先行

MDN 這套材料最值得學的，不是某個標籤、選擇器或 JavaScript API 的寫法，而是以下五個長期有效的程序設計觀念：

1. **Web 程式不是單一 JavaScript 程式，而是多個引擎共同解釋的系統。** HTML 建立內容與語義，CSS 經由層疊和佈局演算法求出呈現結果，JavaScript 在事件與非同步工作中改變狀態，瀏覽器再建立 DOM、渲染樹及無障礙樹。設計時若只看 JavaScript，會漏掉系統大部分既有能力。[MDN：瀏覽器如何載入網站](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Getting_started/Web_standards/How_browsers_load_websites)
2. **原生語義是最便宜、最可靠的抽象。** `<button>`、`<a>`、`<form>`、標題與地標元素同時提供結構、鍵盤行為、可存取名稱、預設互動和工具可理解性。用 `<div>` 重造控制項，等於主動承擔瀏覽器已經解決的狀態機與無障礙責任。[MDN：HTML 與無障礙](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Core/Accessibility/HTML)
3. **前端是分散式系統的一端。** 網路請求可能延遲、失敗、亂序或成功傳輸一個業務失敗；使用者也可以繞過前端驗證。所有跨網路資料都要視為不可信，所有非同步流程都要明確定義成功、失敗、取消、過期及重試語義。[MDN：從伺服器取得資料](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Core/Scripting/Network_requests)、[MDN：站點安全](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Extensions/Server-side/First_steps/Website_security)
4. **可用性、無障礙、效能與安全不是完成後再加的品質層。** 它們會反過來決定 HTML 結構、路由方式、元件 API、資料流、錯誤呈現和測試矩陣；太晚處理通常意味著重寫。[MDN：無障礙](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Core/Accessibility)、[MDN：Web 效能](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Extensions/Performance)
5. **框架是成本交換，不是能力來源。** 框架用執行時、工具鏈與抽象成本換取一致的元件、狀態和路由模型；需求簡單時，原生 Web 平台或靜態生成往往更可靠。即使用框架，最終使用者仍然只與 HTML、CSS、URL、焦點和瀏覽器歷史互動。[MDN：客戶端框架介紹](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Core/Frameworks_libraries/Introduction)

### 整體評價

- **作為入門到中階的地基：優秀。** 課程刻意把目標定為從「萌新」走到「舒適（comfortable）」，而非從萌新直接成為專家（expert）；它把語義、無障礙、測試、效能與工具納入核心視野，比只教框架 API 的課程更耐久。[MDN：關於學習 Web 開發](https://developer.mozilla.org/zh-CN/docs/Learn_web_development#%E5%85%B3%E4%BA%8E%E5%AD%A6%E4%B9%A0_web_%E5%BC%80%E5%8F%91)
- **作為生產工程手冊：不完整。** 課程沒有系統建立授權模型、資料庫約束與交易、API 版本與冪等性、可觀測性、部署回滾、容量規劃、供應鏈治理等完整方法。這不是教材失敗，而是其自述範圍；不要把完成課程誤認為具備獨立設計生產系統的充分條件。[MDN Curriculum 模組範圍](https://developer.mozilla.org/en-US/curriculum/)
- **中文版本可讀，但不應成為唯一權威。** 中文頁明示由社群翻譯，各章更新時間不一；遇到 API 行為、相容性或安全細節，應切換英文最新版並核對規範與 MDN 參考頁，而不是只依賴教學範例。[MDN：翻譯內容說明](https://developer.mozilla.org/zh-CN/docs/MDN/Community/Contributing/Translated_content)

### 重要版本警告：中文安全章有兩項過時建議

這不是一般翻譯措辭差異，而是可能直接導致錯誤安全設計的內容差異：

1. **密碼定期輪換。** 中文〈站點安全〉仍寫有「當密碼頻繁更換時鼓勵更加健壯的密碼」；現行英文頁已刪除「頻繁更換」要求，只建議強密碼與多因素認證。[中文 MDN：關鍵資訊](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Extensions/Server-side/First_steps/Website_security#%E4%B8%80%E4%BA%9B%E5%85%B3%E9%94%AE%E4%BF%A1%E6%81%AF)、[英文 MDN：Key messages](https://developer.mozilla.org/en-US/docs/Learn_web_development/Extensions/Server-side/First_steps/Website_security#key_messages) 現行 NIST SP 800-63B-4 更明確要求：**不得要求使用者週期性更改密碼**；只有在有證據顯示驗證器已遭入侵時才強制更改。產品政策應採 NIST 現行規則，並配合已洩漏/常見密碼 blocklist、密碼管理器與適當多因素驗證，而不是任意 30/60/90 天輪換。[NIST SP 800-63B-4 §3.1.1.2](https://pages.nist.gov/800-63-4/sp800-63b/authenticators/#passwordver)
2. **以轉義作為 SQL injection 主要防禦。** 中文頁的主要示例仍教導轉義 SQL 特殊字元；現行英文 MDN 已改成「最佳實務是參數化查詢（prepared statements）」，令資料與 SQL 程式碼分離，並建議使用安全封裝的 ORM/查詢 API。[中文 MDN：SQL 注入](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Extensions/Server-side/First_steps/Website_security#sql_%E6%B3%A8%E5%85%A5)、[英文 MDN：SQL injection](https://developer.mozilla.org/en-US/docs/Learn_web_development/Extensions/Server-side/First_steps/Website_security#sql_injection) OWASP 也把 prepared statements/parameterized queries 列為首要防禦，並把「轉義所有使用者輸入」標為強烈不建議；只有無法使用 bind parameter 的結構位置才另做 allow-list 驗證。[OWASP：SQL Injection Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html)

因此，本報告引用中文安全章是為了說明威脅與信任邊界；涉及具體控制措施時，以現行英文 MDN、NIST 及 OWASP 的上述內容為準。這也具體證明了「翻譯頁不能成為唯一安全基線」。

## 二、正確的總體心智模型

### 2.1 把頁面看成四棵彼此關聯的樹

工程上可把瀏覽器內的頁面理解為至少四種結構：

1. **DOM 樹**：由 HTML 解析產生，是內容、節點身份和父子關係的來源。
2. **樣式/渲染結構**：CSS 規則經來源、層、重要性、優先級、範圍和順序決議後，參與佈局與繪製。
3. **無障礙樹**：由語義、角色、名稱、狀態等資訊生成，供輔助技術使用。
4. **應用狀態圖**：由 JavaScript、URL、表單和遠端資料共同構成；它不是瀏覽器自動替你建好的，需要程式明確維護。

前三者在 MDN 的瀏覽器載入與無障礙章節中有直接說明；第四者是將事件、框架和網路章節合併後得到的工程模型。[MDN：瀏覽器如何載入網站](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Getting_started/Web_standards/How_browsers_load_websites)、[MDN：事件介紹](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Core/Scripting/Events)、[MDN：客戶端框架介紹](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Core/Frameworks_libraries/Introduction)

**關鍵不變量：** 視覺結果、DOM 語義、無障礙狀態和應用狀態必須描述同一事實。例如按鈕視覺上顯示「已展開」時，對應內容必須真的可見，鍵盤焦點必須可達，`aria-expanded` 也必須同步。若這四者可各自漂移，程式一定會出現「看起來對、實際上錯」的狀態。

### 2.2 把瀏覽器看成受限制、具並行來源的事件系統

- JavaScript 通常在主執行緒上與使用者互動及渲染工作競爭；長時間同步工作會讓頁面無法回應。計算密集工作可移到 Worker，但 Worker 與主執行緒之間以訊息通訊，不能直接操作 DOM。[MDN：非同步 JavaScript 簡介](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Extensions/Async_JS/Introducing)、[MDN：Worker 簡介](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Extensions/Async_JS/Introducing_workers)
- 事件不是單純「呼叫函式」；它具有目標、當前處理者、捕獲、冒泡、預設行為與傳播控制。大型 UI 若不了解傳播模型，會產生重複觸發、誤關閉彈窗或難以清理的監聽器。[MDN：事件冒泡](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Core/Scripting/Event_bubbling)
- 網路、計時器、使用者輸入與 Worker 都可能在不同時間送回結果。**回傳順序不等於發出順序**；程式需要用請求識別、版本號或取消訊號保證舊結果不能覆蓋新狀態。取消機制可使用 `AbortController`；「最後一次意圖勝出」則是應用必須自行維護的不變量。[MDN：AbortController](https://developer.mozilla.org/zh-CN/docs/Web/API/AbortController)

## 三、HTML：先定義資訊模型，再定義外觀

### 3.1 語義就是公開介面

HTML 元素不是裝飾標籤，而是頁面向瀏覽器、搜尋引擎、測試工具、輔助技術和其他開發者提供的介面。`<main>`、`<nav>`、`<article>`、`<section>`、`<header>`、`<footer>` 表達區域職責；標題表達文件層級；列表表達集合；表格表達二維資料關係。[MDN：建立文件與網站結構](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Core/Structuring_content/Structuring_documents)

實務規則：

- 先寫一個**沒有 CSS 仍然順序合理、沒有 JavaScript 仍然可理解**的文件，再加入呈現和增強行為。這是漸進增強的最小形式。[MDN：HTML 與無障礙](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Core/Accessibility/HTML)
- `<div>` 和 `<span>` 只應在沒有合適語義或純粹需要分組時使用。大量無語義容器會讓維護者無法從標記判斷職責，也迫使 CSS、JavaScript 與測試依賴脆弱的結構位置。[MDN：無語義包裝器](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Core/Structuring_content/Structuring_documents#%E6%97%A0%E8%AF%AD%E4%B9%89%E5%8C%85%E8%A3%85%E5%99%A8)
- 不要用表格做版面；表格只表示資料關係。資料表應有標題和可辨識的列/欄標頭，讓輔助技術能把資料格與標頭關聯。[MDN：表格無障礙](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Core/Structuring_content/Table_accessibility)
- 圖片的 `alt` 不是檔案描述，而是圖片在當前任務中的**文字替代**。資訊圖需要傳達等價資訊；純裝飾圖應用空 `alt=""`，避免被重複朗讀。[MDN：替代文字](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Core/Accessibility/HTML#%E6%9B%BF%E4%BB%A3%E6%96%87%E6%9C%AC)
- `id` 在文件樹中必須唯一；它是錨點、標籤關聯、ARIA 引用和腳本查找的身份鍵。重複 `id` 不是純風格問題，而是破壞身份不變量。[WHATWG HTML：`id`](https://html.spec.whatwg.org/multipage/dom.html#the-id-attribute)

### 3.2 原生控制項優於自製控制項

- 用 `<button>` 表示動作、用 `<a href>` 表示導航。兩者的鍵盤、焦點、語義與預設行為不同，不能只因視覺相似而互換。[MDN：HTML 與無障礙的 UI 控制項](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Core/Accessibility/HTML#ui_%E6%8E%A7%E4%BB%B6)
- 自訂表單控制項意味著你要自行實作焦點、鍵盤操作、角色、狀態、名稱、錯誤提示和跨瀏覽器互動。除非原生控制項無法滿足必要需求，否則這筆成本通常不值得。[MDN：如何建立自訂表單控制項](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Extensions/Forms/How_to_build_custom_form_controls)
- ARIA 的用途是補充缺失語義，不是修補錯誤 HTML 的萬用貼紙。W3C 的第一條 ARIA 使用規則是：若原生 HTML 元素已具有所需語義和行為，就優先使用它。[W3C：Using ARIA](https://www.w3.org/TR/using-aria/)
- `role` 是行為承諾，不是行為實作。`role="button"` 只改變無障礙樹中的角色，不會自動提供 Tab 焦點、Enter/Space 啟動、禁用狀態或焦點外觀；開發者必須完成與該角色一致的全部互動。[W3C ARIA APG：Read Me First](https://www.w3.org/WAI/ARIA/apg/practices/read-me-first/)

**關鍵不變量：** 每個互動控制項都必須有可存取名稱、可由所需輸入方式操作、具可見焦點，而且其公開狀態與視覺狀態一致。

### 3.3 HTML 是容錯的，但不等於錯誤無害

瀏覽器會修復某些不合法標記，因此錯誤 HTML 可能「看得到」。但修復後的 DOM 可能與原始碼不同，進而破壞 CSS 選擇器、事件委派、表單關聯和無障礙樹。應檢查解析後 DOM，並把 HTML 驗證納入品質檢查，而不是用視覺結果判定正確。[MDN：除錯 HTML](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Core/Structuring_content/Debugging_HTML)、[WHATWG：HTML parsing](https://html.spec.whatwg.org/multipage/parsing.html)

## 四、CSS：把它當作約束求解系統，而非逐行指令

### 4.1 層疊是架構，不是麻煩

CSS 衝突由來源與重要性、層、優先級、作用域接近度及來源順序共同決定；只有前面的判定相同時，後面的判定才有機會生效。[MDN：層疊、優先級與繼承](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Core/Styling_basics/Handling_conflicts)、[W3C CSS Cascade Level 6](https://www.w3.org/TR/css-cascade-6/)

工程含義：

- 選擇器越具體，覆寫成本越高。用低優先級的類別和明確的元件邊界維持可組合性；不要以 ID、深層後代選擇器和 DOM 路徑建立日後難以拆除的耦合。
- `!important` 會改變正常層疊順序，而且同層競爭往往只能再加一個 `!important`；它應是有邊界的例外，例如明確的工具類或使用者覆寫，而非一般修錯手段。[MDN：`!important`](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Core/Styling_basics/Handling_conflicts#!important)
- 第三方、基礎、元件、工具與覆寫樣式可用 `@layer` 明確排序，讓「誰可覆寫誰」成為架構契約，而不是依賴載入偶然性。[MDN：級聯層](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Core/Styling_basics/Cascade_layers)
- 繼承適合承載字體、文字顏色等共同語境；尺寸、邊框和間距通常不繼承。設計 token 可在高層透過自訂屬性傳遞，但元件仍要為缺值和覆寫定義合理預設。[MDN：理解繼承](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Core/Styling_basics/Handling_conflicts#%E7%90%86%E8%A7%A3%E7%BB%A7%E6%89%BF)

**關鍵不變量：** 任一元件的樣式結果應能從有限且明確的層、token 與元件規則推導，不應依賴整個頁面的 DOM 深度或不透明的載入順序。

### 4.2 先接受正常流，再選佈局工具

- 正常流是可靠預設；Flexbox 適合一維排列與內容分配，Grid 適合二維列欄關係，定位適合脫離或偏移正常流的特殊元素。使用絕對定位建構整頁，會讓內容增長、縮放和翻譯變成例外處理。[MDN：CSS 佈局介紹](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Core/CSS_layout/Introduction)
- 響應式設計不是幾個固定裝置寬度，而是讓內容、容器和可用空間共同決定重排。斷點應在內容開始失效之處出現，而不是追逐某款手機尺寸。[MDN：響應式設計](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Core/CSS_layout/Responsive_Design)
- 行動頁需要正確的 viewport 宣告，例如 `width=device-width, initial-scale=1`；否則行動瀏覽器可能使用虛擬寬視口，令 media query 與真實可用寬度不一致。[MDN：viewport meta](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Core/CSS_layout/Responsive_Design#%E8%A7%86%E5%8F%A3%E5%85%83%E6%A0%87%E7%AD%BE)
- 文字不可只以 `vw` 縮放，否則瀏覽器縮放可能無法有效放大文字；保留 `rem`/`em` 基線，再以有限的流動增幅處理寬螢幕。[MDN：響應式排版](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Core/CSS_layout/Responsive_Design#%E5%93%8D%E5%BA%94%E5%BC%8F%E6%8E%92%E7%89%88)
- 圖片與媒體是替換元素，具有固有尺寸；應預防內容溢出和版面位移，提供合理尺寸約束，並按顯示需求供應適當資源。[MDN：圖片、媒體和表單元素](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Core/Styling_basics/Images_media_forms)、[MDN：多媒體效能](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Extensions/Performance/Multimedia)
- 使用邏輯屬性和彈性佈局可降低由文字方向、語言長度和書寫模式造成的假設。把 `left/right`、固定高度和單行文字當成普遍真理，國際化時必然破裂。[MDN：處理不同文字方向](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Core/Styling_basics/Handling_different_text_directions)

盒模型與內容容量還有兩個常被忽略的不變量：專案應統一 `box-sizing` 策略，避免新增 padding/border 意外擴大宣告寬度；而固定高度配合 `overflow: hidden` 可能在翻譯、字體放大、動態資料和長單字下靜默截斷資訊。只有「截斷本身是產品規則」且有完整內容替代入口時，才應隱藏溢出。[MDN：盒模型](https://developer.mozilla.org/en-US/docs/Learn_web_development/Core/Styling_basics/Box_model)、[MDN：溢出內容](https://developer.mozilla.org/en-US/docs/Learn_web_development/Core/Styling_basics/Overflow)

### 4.3 CSS 除錯應查「計算原因」

不要只反覆改數值。應在開發者工具中依序查：元素是否匹配、宣告是否被覆寫、計算值為何、盒模型尺寸、格式化上下文、包含塊、溢出和堆疊上下文。CSS 問題通常是錯誤心智模型，不是少一個魔法數字。[MDN：除錯 CSS](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Core/Styling_basics/Debugging_CSS)

## 五、JavaScript：資料、狀態、純計算與副作用要分開

### 5.1 變數不只是容器，而是可變狀態的入口

- 預設用 `const` 表示綁定不重新指定；只有確實需要狀態轉移時才用 `let`。這不能讓物件不可變，但能縮小「哪個名稱可能被改寫」的推理範圍。[MDN：變數](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Core/Scripting/Variables)
- 命名應表達業務概念、單位和狀態，而不是實作形狀。例如 `timeoutMs` 比 `t` 更能防止單位錯誤，`isSubmitting` 比 `flag` 更能限制合法狀態。
- 不要用彼此獨立的布林值表示互斥狀態。`isLoading/isSuccess/isError` 可能同時為真；`status: "idle" | "loading" | "success" | "error"` 更接近真正狀態機。這是從 MDN 條件、事件和 Promise 狀態模型延伸出的工程做法。[MDN：條件陳述式](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Core/Scripting/Conditionals)、[MDN：Promise](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Promise)

### 5.2 函式邊界應揭露契約

每個非平凡函式都應能回答：接受什麼、回傳什麼、可能失敗嗎、會修改什麼、是否依賴時間/DOM/網路、可否重複呼叫。將純資料轉換與 DOM、儲存、網路等副作用分開，能讓核心規則在沒有瀏覽器環境時測試，也能避免 UI 事件處理器膨脹成整個應用。[MDN：建立自己的函式](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Core/Scripting/Build_your_own_function)

**建議邊界：**

```text
事件/網路輸入 -> 驗證與正規化 -> 純領域運算 -> 狀態轉移 -> DOM/網路輸出
```

每個箭頭都應有錯誤契約；尤其不要讓解析失敗、缺欄位或無效狀態默默變成 `undefined`，再在遠端位置爆炸。

### 5.3 錯誤分成語法、執行時與邏輯錯誤

MDN 明確區分語法錯誤與邏輯錯誤。前者通常由解析器或執行時直接指出；後者能正常執行卻產生錯誤結果，只能藉由不變量、案例、斷點、日誌和測試發現。[MDN：查找並解決 JavaScript 錯誤](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Core/Scripting/What_went_wrong)

實務上還要區分：

- **預期的領域拒絕**：例如餘額不足、欄位重複；需要穩定錯誤碼與可理解訊息。
- **暫時性基礎設施失敗**：例如逾時、斷線、503；可能允許有界重試。
- **程式缺陷**：例如不可能狀態、空值違反契約；應記錄足夠上下文並快速暴露，而非假裝成功。

**關鍵不變量：** 每個錯誤要在仍保有足夠上下文的邊界被處理；不能處理就傳遞，不能靜默吞掉。

## 六、事件驅動 UI：生命週期和傳播是核心

### 6.1 使用 `addEventListener()`，避免內聯事件

`addEventListener()` 支援多個處理器、選項和可控移除；內聯 `onclick` 混合結構與行為、難以擴展，也會與嚴格內容安全政策衝突。[MDN：事件介紹](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Core/Scripting/Events)

注意事項：

- 監聽器是資源。元件卸載、頁面切換或任務取消時，應以 `removeEventListener()` 或 `AbortSignal` 清理，否則閉包可能保留資料並重複回應。[MDN：移除監聽器](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Core/Scripting/Events#%E7%A7%BB%E9%99%A4%E7%9B%91%E5%90%AC%E5%99%A8)
- `event.target` 是最初觸發節點，`event.currentTarget` 是目前執行處理器的節點。事件委派若混淆兩者，點擊子圖示時常會取錯資料。[MDN：事件委派](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Core/Scripting/Event_bubbling#%E4%BA%8B%E4%BB%B6%E5%A7%94%E6%89%98)
- `preventDefault()` 阻止瀏覽器預設動作；`stopPropagation()` 阻止傳播。兩者目的不同。過度停止傳播會破壞外層委派、分析或可組合元件。[MDN：阻止預設行為](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Core/Scripting/Events#%E9%98%BB%E6%AD%A2%E9%BB%98%E8%AE%A4%E8%A1%8C%E4%B8%BA)、[MDN：事件冒泡](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Core/Scripting/Event_bubbling)
- 用事件委派處理大量或動態子項，可減少監聽器數量並避免新節點忘記綁定；處理器仍需用 `closest()` 或等價邏輯驗證目標確實屬於預期控制項。

### 6.2 事件處理器要短、可重入、能防重複提交

事件可能密集或重複發生。處理器應快速讀取意圖、轉交應用邏輯並立即返回；不要在其中做長同步工作。對提交、付款、建立資源等動作，UI 禁用只能改善體驗，真正的「只執行一次」必須由伺服器端冪等鍵或資料約束保證。[MDN：非同步 JavaScript 簡介](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Extensions/Async_JS/Introducing)、[IETF HTTP Semantics：冪等方法](https://www.rfc-editor.org/rfc/rfc9110.html#name-idempotent-methods)

## 七、非同步與網路：把時間納入資料模型

### 7.1 Promise 是結果佔位，不是背景執行緒

Promise 表示一個操作最終 fulfilled 或 rejected 的結果，並提供組合與錯誤傳播；它本身不保證工作在另一執行緒，也不會自動取消底層操作。[MDN：如何使用 Promise](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Extensions/Async_JS/Promises)、[ECMAScript：Promise Objects](https://tc39.es/ecma262/multipage/control-abstraction-objects.html#sec-promise-objects)

實務規則：

- Promise 鏈中的每一步要 `return` 新值或新 Promise，否則外層無法等待，也會讓錯誤脫離預期鏈。
- 在鏈尾統一捕捉可處理的失敗，但不要把錯誤轉成假成功；若呼叫者仍需知道失敗，處理後要重新拋出或回傳明確結果。
- `Promise.all()` 適合「全部成功才有意義」且彼此獨立的工作；會在任一拒絕時快速拒絕。若需要收集每項結果，使用 `Promise.allSettled()`；不要把不同業務語義硬塞成同一組合。[MDN：Promise 並行](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Extensions/Async_JS/Promises#%E5%90%88%E5%B9%B6%E4%BD%BF%E7%94%A8%E5%A4%9A%E4%B8%AA_promise)
- `async/await` 只改變寫法，不消除競態、取消、逾時和部分失敗。連續 `await` 會序列化工作；沒有資料依賴的請求可先啟動後共同等待。

### 7.2 `fetch()` 的成功不等於業務成功

`fetch()` 在收到 HTTP 回應後會履行 Promise，即使狀態是 404 或 500；程式必須檢查 `response.ok`/`status`，再解析內容。解析 JSON 本身也是非同步且可能失敗。[MDN：如何使用 Promise 的 fetch 範例](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Extensions/Async_JS/Promises#%E4%BD%BF%E7%94%A8_fetch_api)

一個完整請求狀態至少應區分：

```text
idle -> loading -> success
               -> HTTP error
               -> network error
               -> parse/schema error
               -> aborted/stale
```

另外仍要有**業務錯誤**，例如 HTTP 200 中回傳操作未獲授權或資料版本衝突。前後端契約至少要定義：狀態碼、內容類型、成功 schema、錯誤碼、可顯示訊息、可否重試、追蹤識別碼與相容性策略。[MDN：Fetch API](https://developer.mozilla.org/zh-CN/docs/Web/API/Fetch_API)、[IETF HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110.html)

### 7.3 取消、過期與競態必須顯式處理

典型搜尋框競態：使用者先搜 A、再搜 B；B 先返回，A 後返回並覆蓋畫面。解法不是假設網路有序，而是維持下列任一不變量：

- 新查詢開始時用 `AbortController` 取消舊請求；或
- 每次請求帶遞增版本，只有目前版本可提交狀態；或
- 把查詢鍵納入快取/狀態身份，只允許結果更新同一鍵。

取消只表示呼叫端不再需要結果，不保證伺服器已停止副作用；有寫入行為時仍需伺服器冪等性與交易保護。[MDN：AbortController](https://developer.mozilla.org/zh-CN/docs/Web/API/AbortController)、[MDN：Fetch `signal`](https://developer.mozilla.org/zh-CN/docs/Web/API/RequestInit#signal)

### 7.4 Worker 的邊界是資料訊息

Worker 適合 CPU 密集、可資料化的工作；它不能直接操作 DOM，資料在執行緒間以訊息傳遞。設計 Worker 時應先定義訊息 schema、工作 ID、進度、取消、錯誤和版本，而不是把任意物件到處傳。[MDN：Worker 簡介](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Extensions/Async_JS/Introducing_workers)、[WHATWG HTML：Workers](https://html.spec.whatwg.org/multipage/workers.html)

## 八、表單與資料完整性：UX 驗證不等於信任邊界

### 8.1 使用瀏覽器約束，但伺服器必須重新驗證

HTML 的 `required`、`type`、`min/max`、`minlength/maxlength` 和 `pattern` 能提供即時回饋與一致的約束 API；優先使用它們，再用 JavaScript 補足跨欄位或業務規則。[MDN：表單資料驗證](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Extensions/Forms/Form_validation)

但使用者可以關閉 JavaScript、修改 DOM、直接呼叫 API 或重放請求，所以：

- 客戶端驗證的目的：快速回饋、減少無效往返、協助正確輸入。
- 伺服器驗證的目的：保護資料與業務不變量，是唯一可信的執行邊界。
- 資料庫約束的目的：在併發和程式缺陷下仍保證唯一性、外鍵、非空與檢查條件。

MDN 明確指出伺服器不能信任客戶端驗證。[MDN：傳送與擷取表單資料](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Extensions/Forms/Sending_and_retrieving_form_data)

**關鍵不變量：** 同一核心業務規則可以在多層提供回饋，但必須在最接近資料所有權、無法繞過的一層強制執行。

### 8.2 表單是 HTTP 契約，不只是 UI

- `name` 決定提交鍵，`value` 決定值，`action` 決定目的地，`method` 決定傳輸語義，`enctype` 決定編碼。任何一項不明確都可能令後端收到不同資料。[MDN：傳送表單資料](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Extensions/Forms/Sending_and_retrieving_form_data)
- 表單內每個 `<button>` 都應明確寫出 `type`；省略時通常預設為 submit，工具列、增減器或開啟對話框按鈕可能意外送出表單。[MDN：`<button>`](https://developer.mozilla.org/zh-CN/docs/Web/HTML/Reference/Elements/button)
- 後端必須定義「缺少 key」的語義。沒有 `name`、被 disabled、未勾選的 checkbox/radio 等控制項不會按一般成功控制項提交；不能不經契約就把缺值一律解讀成 `false`、空字串或「維持原值」。[WHATWG：Constructing the entry list](https://html.spec.whatwg.org/multipage/form-control-infrastructure.html#constructing-the-entry-list)
- 敏感資訊不應以 GET 查詢字串提交，因為 URL 會出現在歷史、日誌、分析和引用來源中；修改狀態的操作也不應使用安全方法 GET。[MDN：GET 與 POST](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Extensions/Forms/Sending_and_retrieving_form_data)、[IETF：Safe Methods](https://www.rfc-editor.org/rfc/rfc9110.html#name-safe-methods)
- 檔案上傳要限制大小、數量、類型和儲存位置；不能信任副檔名或客戶端 `Content-Type`。上傳內容最好與應用執行來源隔離，並由伺服器重新命名與授權存取。[MDN：傳送檔案與安全](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Extensions/Forms/Sending_and_retrieving_form_data#%E7%89%B9%E6%AE%8A%E6%A1%88%E4%BE%8B%EF%BC%9A%E5%8F%91%E9%80%81%E6%96%87%E4%BB%B6)
- 錯誤訊息要與欄位程式化關聯，指出如何修正，且不要只靠顏色。提交失敗後應保留可安全保留的輸入，避免使用者重填。

### 8.3 安全規則依輸出語境而定

「清理輸入一次」不是完整安全模型。同一字串放入 HTML 文字、HTML 屬性、URL、CSS、JavaScript 或 SQL，需要不同編碼/參數化策略。不可把使用者字串拼成可執行 HTML、SQL、HTTP header、郵件 header、命令或檔案路徑。[MDN：站點安全](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Extensions/Server-side/First_steps/Website_security)、[MDN：XSS](https://developer.mozilla.org/zh-CN/docs/Web/Security/Attacks/XSS)

最低安全不變量：

1. 任何來自瀏覽器的 URL 參數、表單、JSON、header、cookie 和上傳檔都不可信。
2. 認證只回答「你是誰」；每次資源操作仍須授權判定「你能否對這個物件做這件事」。
3. 使用 HTTPS，敏感 cookie 應採合適的 `Secure`、`HttpOnly`、`SameSite` 屬性；狀態變更需 CSRF 防護。[MDN：Cookie 安全](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Guides/Cookies#security)
4. SQL 必須透過框架查詢 API 或參數化查詢，不得字串拼接使用者輸入；HTML 輸出要按語境編碼。
5. 只收集、保存和顯示完成任務所必需的資料，降低外洩後果。[MDN：站點安全的關鍵資訊](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Extensions/Server-side/First_steps/Website_security#%E4%B8%80%E4%BA%9B%E5%85%B3%E9%94%AE%E4%BF%A1%E6%81%AF)

## 九、無障礙：它是系統正確性的一部分

W3C WCAG 以可感知、可操作、可理解、穩健（POUR）組織成功準則。這比「加 alt」更完整：輸入方式、焦點順序、對比、縮放、錯誤識別、動態內容公告和語義解析都屬於系統行為。[W3C：WCAG 2.2](https://www.w3.org/TR/WCAG22/)

### 9.1 實作規則

- 保持合理標題層級和地標，讓使用者可跳過重複內容、快速理解頁面結構。[MDN：HTML 與無障礙](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Core/Accessibility/HTML)
- 所有核心流程必須可只用鍵盤完成；焦點指示不可被無替代地移除。視覺順序與 DOM/焦點順序應一致，避免看見的下一步不是鍵盤的下一步。[W3C：Focus Order](https://www.w3.org/WAI/WCAG22/Understanding/focus-order.html)
- 顏色不能是唯一訊息通道；文字需足夠對比，介面在放大和文字重排時仍可使用。[W3C：Use of Color](https://www.w3.org/WAI/WCAG22/Understanding/use-of-color.html)、[W3C：Reflow](https://www.w3.org/WAI/WCAG22/Understanding/reflow.html)
- 動態錯誤、載入完成、儲存成功等狀態若不移動焦點，應透過適當 live region 通知輔助技術，但避免頻繁公告造成噪音。[MDN：ARIA live regions](https://developer.mozilla.org/zh-CN/docs/Web/Accessibility/ARIA/Guides/Live_regions)
- SPA 路由不會自動觸發完整頁面導航的標題宣告與焦點重設。每次視圖切換要更新文件標題、管理焦點，並保持 URL、返回/前進與深層連結可用。[MDN：框架頁面的無障礙](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Core/Frameworks_libraries/Introduction#%E6%A1%86%E6%9E%B6%E9%A9%B1%E5%8A%A8%E9%A1%B5%E9%9D%A2%E7%9A%84%E6%97%A0%E9%9A%9C%E7%A2%8D)

### 9.2 測試原則

自動化工具能找出缺少名稱、無效 ARIA 和部分對比問題，但無法判定替代文字是否等價、焦點流程是否合理、錯誤是否易懂。至少要結合語義檢查、鍵盤操作、縮放/重排、高對比或偏好設定，以及實際螢幕閱讀器冒煙測試。[MDN：無障礙疑難排解](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Core/Accessibility/Accessibility_troubleshooting)

## 十、效能：先定義使用者體驗，再談微優化

### 10.1 效能是載入、回應與穩定性的共同結果

HTML 解析會發現並請求 CSS、JavaScript、字型、圖片等資源；資源的大小、優先級、相依瀑布和主執行緒工作共同決定何時看得見、何時可互動。大量 JavaScript 同時增加下載、解析、編譯與執行成本。[MDN：瀏覽器如何載入網站](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Getting_started/Web_standards/How_browsers_load_websites)、[MDN：HTML 效能](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Extensions/Performance/HTML)

實務順序：

1. 先定義關鍵使用流程與真實使用者/裝置/網路條件。
2. 建立基線並量測，而不是憑感覺優化。[MDN：測量效能](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Extensions/Performance/Measuring_performance)
3. 先移除不必要資源與工作，再壓縮、延遲或快取。
4. 設效能預算，例如初始 JavaScript、關鍵圖片、字型數量和關鍵流程延遲，並在 CI 或監控中防止回退。
5. 同時測試冷快取、熱快取、慢網路、低階 CPU 和真實資料量。

### 10.2 高槓桿做法

- 讓 HTML 儘早帶出主要內容；避免內容完全依賴 JavaScript 建構。這同時改善可靠性、SEO、無障礙和啟動成本。[MDN：框架的大型程式碼庫與抽象](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Core/Frameworks_libraries/Introduction#%E5%A4%A7%E5%9E%8B%E4%BB%A3%E7%A0%81%E5%BA%93%E4%B8%8E%E6%8A%BD%E8%B1%A1)
- 明確選擇 script 載入契約：普通 classic script 會阻塞後續 HTML 解析；`defer` 可平行下載並在解析後按文件順序執行；`async` 下載完成即執行，彼此順序不可依賴；module 具有 defer 類行為。不能只為「快」而破壞依賴次序。[MDN：`<script>`](https://developer.mozilla.org/zh-CN/docs/Web/HTML/Reference/Elements/script)
- 選擇合適圖片格式與尺寸，使用響應式圖片；提供寬高或穩定比例，避免載入後版面跳動。[MDN：多媒體效能](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Extensions/Performance/Multimedia)
- 字型會增加網路與文字呈現風險；限制字型家族/字重，提供合理 fallback，並按內容需求載入。[MDN：Web 字型](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Core/Text_styling/Web_fonts)
- 動畫優先使用瀏覽器能高效處理的屬性，並尊重 `prefers-reduced-motion`；感知效能不能以造成不適為代價。[MDN：`prefers-reduced-motion`](https://developer.mozilla.org/zh-CN/docs/Web/CSS/@media/prefers-reduced-motion)
- 對無法立即完成的動作提供即時、誠實的進度或狀態回饋；不要用假的進度掩蓋無界等待。[MDN：感知效能](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Extensions/Performance/Perceived_performance)

**關鍵不變量：** 效能改動必須由與使用者目標相關的量測證明，且不能破壞可讀性、正確性、無障礙或安全。

## 十一、框架與元件：先定邊界，後選工具

### 11.1 何時框架值得使用

框架在大量互動狀態、可重用元件、複雜路由、團隊一致性和成熟工具整合下能降低認知負擔。若只是少量頁面和互動，框架執行時、建置與升級成本可能超過收益。[MDN：框架存在原因與過度工程](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Core/Frameworks_libraries/Introduction#%E4%BD%BF%E7%94%A8%E6%A1%86%E6%9E%B6%E7%9A%84%E6%B3%A8%E6%84%8F%E4%BA%8B%E9%A1%B9)

選型至少評估：

- 使用者瀏覽器、裝置和無 JavaScript/慢 JavaScript 情況；
- 團隊現有能力、學習與招募成本；
- 社群、文件、維護節奏與長期可持續性；
- SSR/SSG、路由、資料載入、測試和無障礙支援；
- 產物大小、執行成本、升級和退出成本。

MDN 特別提醒瀏覽器支援、DSL、社群及文件是框架選擇因素，且框架不是所有問題的解法。[MDN：如何選擇框架](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Core/Frameworks_libraries/Introduction#%E5%A6%82%E4%BD%95%E9%80%89%E6%8B%A9%E4%B8%80%E4%B8%AA%E6%A1%86%E6%9E%B6)

### 11.2 好元件的判準

- 以單一使用者/領域責任劃界，不以「看起來能拆」為唯一標準。
- API 表達必要變體和事件，不暴露內部 DOM 結構。
- 狀態有單一所有者；衍生資料由來源計算，不維護第二份可漂移狀態。
- 預設輸出語義 HTML，支援鍵盤、焦點與可存取名稱。
- 非同步元件有 loading、empty、error、success、stale/cancelled 狀態。
- 移除元件會清理事件、計時器、訂閱和請求。

MDN 將元件定義為可維護、可重用、可互相通訊的區塊；上述判準是把這一定義轉成可驗證契約。[MDN：元件化](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Core/Frameworks_libraries/Introduction#%E7%BB%84%E4%BB%B6%E5%8C%96)

### 11.3 URL 也是狀態

使用者期待重新整理、深層連結、分享、返回和前進仍能重建同一可導航狀態。重要篩選、分頁、選中資源與視圖身份應按產品需要映射到 URL，而不是只存在記憶體。SPA 若破壞這些能力，就是破壞 Web 的基本介面。[MDN：路由](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Core/Frameworks_libraries/Introduction#%E8%B7%AF%E7%94%B1)

## 十二、伺服器端與資料模型：教學範例之外要補的工程層

MDN 的 Django 介紹清楚展示 URL 路由、視圖、模型、查詢與模板等邊界，這是一個良好起點。[MDN：Django 介紹](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Extensions/Server-side/Django/Introduction)

但真實系統在寫模型前必須先回答：

- 實體身份是什麼？自然鍵還是代理鍵？可否變更？
- 哪些欄位唯一、必填、可空？空字串與 null 是否同義？
- 關係刪除時是限制、級聯、軟刪除還是匿名化？
- 哪些狀態轉移合法？誰有權觸發？可否逆轉？
- 兩個請求同時修改時如何防止遺失更新？
- 一個跨多表操作部分失敗時，哪些變更必須原子提交？
- 重試是否可能重複收款、寄信或建立資料？

這些問題不能只靠 UI 或 ORM 慣例。**不變量應在最接近資料所有權的資料庫約束、交易和服務邊界強制執行**；HTTP 層再把拒絕映射成穩定的錯誤契約。MDN 已指出模型是資料定義和操作的中心，但交易、併發與完整性策略需要另行深入學習。[MDN：Django 模型介紹](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Extensions/Server-side/Django/Introduction#%E5%AE%9A%E4%B9%89%E6%95%B0%E6%8D%AE%E6%A8%A1%E5%9E%8B_models.py)

## 十三、工具鏈、依賴與版本控制

### 13.1 依賴是你交付的一部分

套件可避免重造已解決且充分測試的問題，但每個直接依賴會帶入遞移依賴、版本、漏洞、體積、授權和維護風險。[MDN：軟體套件管理基礎](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Extensions/Client-side_tools/Package_management)

加入依賴前應問：

1. 這是否真是複雜、非核心且已有成熟解法的問題？
2. 原生平台是否已足夠？
3. 套件大小、執行成本、維護頻率、授權和漏洞歷史如何？
4. 若停止維護，替換邊界是否清楚？
5. 只使用很小功能時，能否避免引入整個套件？

實務注意：

- 依賴應安裝在專案本地並以 manifest/lockfile 使建置可重現；不同機器不應偷偷使用不同全域工具版本。[MDN：本地依賴與版本鎖定](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Extensions/Client-side_tools/Package_management#%E4%BB%80%E4%B9%88%E6%98%AF%E8%BD%AF%E4%BB%B6%E5%8C%85%E7%AE%A1%E7%90%86%E5%99%A8%EF%BC%9F)
- 開發建置與生產建置不同；熱重載、未壓縮原始碼等開發功能不應進入生產產物。[MDN：生產建置](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Extensions/Client-side_tools/Package_management#%E4%B8%BA%E7%94%9F%E4%BA%A7%E7%8E%AF%E5%A2%83%E6%9E%84%E5%BB%BA%E6%88%91%E4%BB%AC%E7%9A%84%E4%BB%A3%E7%A0%81)
- 原始碼與部署產物是不同責任邊界：`src` 是可維護輸入，`dist` 是固定命令生成的輸出。不要直接修生成物；CI 應能從乾淨 checkout 重建、測試並發布同一產物。[MDN：完整工具鏈](https://developer.mozilla.org/en-US/docs/Learn_web_development/Extensions/Client-side_tools/Introducing_complete_toolchain)
- 定期審查依賴樹與已知漏洞，但不要把「audit 沒報警」當作供應鏈安全證明。[MDN：漏洞審查](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Extensions/Client-side_tools/Package_management#%E6%BC%8F%E6%B4%9E%E5%AE%A1%E6%9F%A5)
- 升級應在自動測試、產物比較和可回滾前提下進行；語義化版本是發布者承諾，不是相容性的數學證明。

### 13.2 版本控制保存決策，不只是備份

Git 的價值在於可追溯歷史、隔離工作、可控整合與回復，而不是產生 `final_v3` 檔案。[MDN：Git 和 GitHub](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Core/Version_control)

工程做法：

- 每個提交應是一個可理解、可驗證的決策單位；訊息寫「為何」，差異顯示「如何」。
- 分支/PR 應保持範圍小，先通過自動檢查，再由審查者檢查需求、不變量、失敗模式與可維護性。
- 生成物、密鑰和環境專屬狀態不應進入倉庫；可重建的輸出由鎖定工具鏈產生。
- 合併衝突不是機械選邊；它常表示兩個決策同時改變同一不變量，解決後要重新測試行為。

## 十四、測試與除錯：以風險分層，而不是追求數量

### 14.1 先定義支援矩陣

跨瀏覽器測試要從目標使用者、瀏覽器、裝置和無障礙需求開始，再決定測試範圍。不能在所有環境追求像素一致；目標是核心內容和操作可用，並在較強環境提供增強。[MDN：跨瀏覽器測試介紹](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Extensions/Testing/Introduction)

測試層次應對應風險：

- **靜態檢查**：語法、型別、lint、HTML/ARIA 規則，快速阻止明顯缺陷。
- **單元/性質測試**：純領域規則、邊界值與不變量。
- **整合測試**：元件與瀏覽器 API、資料層、服務契約的連接。
- **端到端測試**：少量最關鍵使用流程，包括真正路由、網路和持久化。
- **人工探索**：鍵盤、螢幕閱讀器、視覺重排、慢網路、真實裝置和難以自動判定的體驗。

端到端測試應自治、可重播，不依賴前一測試留下的資料；對非同步 UI 應等待可觀察條件，而不是固定 sleep。固定等待既拖慢測試，也會在較慢環境偶發失敗。[W3C：WebDriver](https://www.w3.org/TR/webdriver2/)

MDN 的課程示範手動、虛擬機、瀏覽器與自動化測試工具；上述分層是把它們按失敗成本組織起來。[MDN：跨瀏覽器測試](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Extensions/Testing)

### 14.2 先用特性檢測與漸進增強

不同瀏覽器可能支援不同功能。應針對所需功能做特性檢測，提供可接受替代，而不是依賴脆弱的 user-agent 字串猜測瀏覽器。修復後必須在原問題環境及其他支援環境回歸測試。[MDN：跨瀏覽器測試工作流](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Extensions/Testing/Introduction#%E8%B7%A8%E6%B5%8F%E8%A7%88%E5%99%A8%E6%B5%8B%E8%AF%95%E7%9A%84%E5%B7%A5%E4%BD%9C%E6%B5%81)、[MDN：特性檢測](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Extensions/Testing/Feature_detection)

### 14.3 除錯循環

1. 取得最小可重現案例和精確環境。
2. 先確認預期行為與實際行為，不急著猜原因。
3. 檢查瀏覽器 console、network、DOM、computed style、accessibility tree 和伺服器日誌。
4. 一次只改一個假設，縮小失敗範圍。
5. 找到根因後加入能防止回歸的測試或約束。
6. 在相鄰環境與流程重測，避免局部修復製造新缺陷。

這與 MDN 的「規劃、開發、測試/查錯、修復/迭代」工作流一致。[MDN：測試工作流](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Extensions/Testing/Introduction#%E8%B7%A8%E6%B5%8F%E8%A7%88%E5%99%A8%E6%B5%8B%E8%AF%95%E7%9A%84%E5%B7%A5%E4%BD%9C%E6%B5%81)

## 十五、核心不變量清單

以下是把全課程濃縮成可在設計審查中逐條詢問的規則：

| 領域 | 不變量 | 主要強制位置 | 常見破壞方式 | 來源 |
|---|---|---|---|---|
| 身份 | 文件內 `id` 唯一；業務實體身份穩定 | HTML 驗證；資料庫鍵 | 重複 DOM id；用可變名稱當主鍵 | [WHATWG `id`](https://html.spec.whatwg.org/multipage/dom.html#the-id-attribute) |
| 語義 | 元素角色與實際操作一致 | HTML 元素/元件 API | 用 `div` 偽裝按鈕；用按鈕做導航 | [MDN HTML 無障礙](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Core/Accessibility/HTML) |
| UI 狀態 | 視覺、DOM、ARIA、應用狀態一致 | 單一狀態所有者 | 各自維護多份布林狀態 | [MDN 無障礙](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Core/Accessibility) |
| 導航 | 可分享 URL 能重建重要視圖 | 路由層 | 狀態只存在記憶體；返回鍵失效 | [MDN 路由](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Core/Frameworks_libraries/Introduction#%E8%B7%AF%E7%94%B1) |
| 非同步 | 過期結果不能覆蓋最新意圖 | 請求/狀態層 | 搜尋結果亂序；卸載後更新 | [MDN AbortController](https://developer.mozilla.org/zh-CN/docs/Web/API/AbortController) |
| 網路 | 傳輸成功、HTTP 成功、解析成功、業務成功分開 | API client/服務邊界 | `fetch` 不檢查 `ok`；吞掉錯誤 | [MDN Promise](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Extensions/Async_JS/Promises) |
| 表單 | 客戶端回饋可繞過，核心規則需伺服器重驗 | 伺服器與資料庫 | 只靠 JavaScript 驗證 | [MDN 表單傳送](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Extensions/Forms/Sending_and_retrieving_form_data) |
| 資料完整性 | 唯一、外鍵、非空與狀態轉移不可因併發失效 | 資料庫約束/交易 | 先查再寫但無唯一約束 | [MDN Django 模型](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Extensions/Server-side/Django/Introduction#%E5%AE%9A%E4%B9%89%E6%95%B0%E6%8D%AE%E6%A8%A1%E5%9E%8B_models.py) |
| 安全 | 所有瀏覽器輸入不可信；每次操作都要授權 | 伺服器邊界 | 信任隱藏欄位、cookie 或前端角色 | [MDN 站點安全](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Extensions/Server-side/First_steps/Website_security) |
| 事件 | 監聽器生命週期不超過擁有者 | 元件/控制器 | 重複綁定、未移除、錯用 target | [MDN 事件](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Core/Scripting/Events) |
| CSS | 覆寫關係可由有限層與低優先級規則推導 | 樣式架構 | ID、深層選擇器、濫用 important | [MDN 層疊](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Core/Styling_basics/Handling_conflicts) |
| 無障礙 | 核心流程可由鍵盤及輔助技術完成 | 元件/流程測試 | 無焦點、名稱、公告或合理順序 | [WCAG 2.2](https://www.w3.org/TR/WCAG22/) |
| 效能 | 優化由真實使用者目標與量測證明 | 效能預算/監控 | 只測開發機；只做微基準 | [MDN 測量效能](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Extensions/Performance/Measuring_performance) |
| 依賴 | 建置可重現且依賴風險可追蹤 | manifest、lockfile、CI | 全域漂移；無審查自動升級 | [MDN 套件管理](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Extensions/Client-side_tools/Package_management) |

## 十六、最常見的陷阱與對策

1. **先選框架，後找問題。** 對策：先寫出使用者流程、資料與互動複雜度，再比較靜態 HTML、漸進增強、SSG/SSR 和 SPA。[MDN：框架注意事項](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Core/Frameworks_libraries/Introduction#%E4%BD%BF%E7%94%A8%E6%A1%86%E6%9E%B6%E7%9A%84%E6%B3%A8%E6%84%8F%E4%BA%8B%E9%A1%B9)
2. **視覺完成即等於功能完成。** 對策：檢查語義、鍵盤、焦點、錯誤、慢網路、重新整理、返回鍵和無 JavaScript 基線。
3. **用更高 CSS 優先級修正架構錯誤。** 對策：先檢查層、所有權和選擇器邊界，再考慮覆寫。[MDN：CSS 層疊](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Core/Styling_basics/Handling_conflicts)
4. **把 `async/await` 當成同步程式。** 對策：畫出同時在途工作、取消、過期和失敗狀態；明確定義誰可提交結果。[MDN：非同步 JavaScript](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Extensions/Async_JS)
5. **只處理網路例外，不處理 HTTP/解析/業務錯誤。** 對策：分層錯誤型別與穩定契約。[MDN：Fetch Promise](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Extensions/Async_JS/Promises#%E4%BD%BF%E7%94%A8_fetch_api)
6. **前端驗證被當作安全控制。** 對策：伺服器重驗，資料庫強制不變量，輸出按語境編碼。[MDN：表單安全](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Extensions/Forms/Sending_and_retrieving_form_data#%E5%B8%B8%E8%A7%81%E7%9A%84%E5%AE%89%E5%85%A8%E9%97%AE%E9%A2%98)
7. **自製控制項只實作滑鼠點擊。** 對策：優先原生元素；必要時把完整鍵盤、焦點、角色、狀態和測試列入完成定義。[W3C：Using ARIA](https://www.w3.org/TR/using-aria/)
8. **響應式等於三個固定寬度。** 對策：以內容失效點、容器和輸入方式測試，避免裝置名驅動。[MDN：響應式設計](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Core/CSS_layout/Responsive_Design)
9. **自動化測試全綠即等於可用。** 對策：加入真實瀏覽器、鍵盤、螢幕閱讀器、慢網路和探索測試。[MDN：跨瀏覽器測試](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Extensions/Testing)
10. **依賴越多開發越快。** 對策：計算遞移成本、產物、漏洞、升級與退出成本，保留替換邊界。[MDN：套件依賴](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Extensions/Client-side_tools/Package_management#%E9%A1%B9%E7%9B%AE%E4%B8%AD%E7%9A%84%E4%BE%9D%E8%B5%96%E9%A1%B9)
11. **用 loading spinner 取代失敗設計。** 對策：每個遠端流程明確設計空、慢、錯、取消、重試和部分完成狀態。
12. **只在單一開發者環境驗證。** 對策：以支援矩陣、可重現建置和 CI 固化環境假設。[MDN：跨瀏覽器工作流](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Extensions/Testing/Introduction#%E8%B7%A8%E6%B5%8F%E8%A7%88%E5%99%A8%E6%B5%8B%E8%AF%95%E7%9A%84%E5%B7%A5%E4%BD%9C%E6%B5%81)

## 十七、MDN 課程沒有充分涵蓋的生產注意事項

以下主題應在完成核心課程後另立學習與設計工作，不能由零散教學範例替代：

- **身份、認證與授權模型**：角色不足以覆蓋物件級、租戶級和動作級權限；需要拒絕優先、最小權限與審計。
- **資料庫完整性與交易**：唯一性、外鍵、交易隔離、鎖、樂觀併發、遷移和備份恢復。
- **API 契約治理**：schema、錯誤碼、版本、分頁、冪等鍵、速率限制、逾時與重試預算。
- **可靠性與可觀測性**：結構化日誌、metrics、trace、SLO、告警、故障降級與事故復盤。
- **交付工程**：CI/CD、環境差異、密鑰管理、資料遷移次序、漸進發布和回滾。
- **供應鏈安全**：來源完整性、鎖檔審查、建置權限、發布簽章和被接管套件風險。
- **隱私與合規**：資料最小化、保存期限、用途限制、刪除/匯出與第三方處理者。
- **大規模前端架構**：跨團隊模組所有權、狀態邊界、設計系統治理、相容遷移和套件發布策略。

這個判斷與 MDN 自述「從萌新到舒適而非專家」一致；也可由正式課程模組清單看出其重點仍是前端核心能力與擴展概覽。[MDN：學習區目標](https://developer.mozilla.org/zh-CN/docs/Learn_web_development#%E5%85%B3%E4%BA%8E%E5%AD%A6%E4%B9%A0_web_%E5%BC%80%E5%8F%91)、[MDN Curriculum](https://developer.mozilla.org/en-US/curriculum/)

## 十八、建議閱讀與實作順序

1. **Web 標準與瀏覽器載入**：先知道程式實際在哪些引擎和網路邊界中運作。[Web standards](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Getting_started/Web_standards)
2. **語義 HTML + 無障礙 HTML**：先建立可工作的資訊與互動基線。[Structuring content](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Core/Structuring_content)、[Accessibility HTML](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Core/Accessibility/HTML)
3. **CSS 層疊、盒模型、正常流、Flex/Grid、響應式**：先理解求值規則，再做視覺系統。[Styling basics](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Core/Styling_basics)、[CSS layout](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Core/CSS_layout)
4. **JavaScript 資料、函式、DOM、事件**：以小型無框架應用練習狀態與副作用分離。[Scripting](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Core/Scripting)
5. **Promise、Fetch、取消與 Worker**：為每個練習加入慢、錯、亂序和取消情境。[Async JavaScript](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Extensions/Async_JS)
6. **表單、驗證、HTTP 與安全**：同時寫出客戶端體驗與伺服器信任邊界。[Forms](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Extensions/Forms)、[Website security](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Extensions/Server-side/First_steps/Website_security)
7. **測試、效能、版本控制與套件管理**：把品質門檻和可重現性加入專案，而不是只完成畫面。[Testing](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Extensions/Testing)、[Performance](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Extensions/Performance)、[Version control](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Core/Version_control)
8. **最後才學框架**：把同一個小應用用框架重寫，比較它真正移除了哪些成本、又引入哪些成本。[Frameworks and libraries](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Core/Frameworks_libraries)

每個階段不只做 happy path；至少加入：空資料、無效輸入、重複操作、網路失敗、慢回應、窄螢幕、鍵盤使用、頁面重新整理與返回鍵。

## 十九、可直接用於專案審查的完成定義

### 問題與資料

- [ ] 使用者、核心任務、成功條件與明確不做的事已寫清楚。
- [ ] 實體身份、唯一性、可空性、關係與合法狀態轉移已定義。
- [ ] 每個不變量都有不能被繞過的強制位置。

### 介面與互動

- [ ] HTML 在無 CSS/JavaScript 時仍有合理結構與可理解內容。
- [ ] 優先使用原生元素；所有控制項有名稱、焦點和鍵盤行為。
- [ ] loading、empty、error、success、取消和重試狀態均有設計。
- [ ] 重要視圖可透過 URL 分享、重新整理並使用返回/前進。

### 非同步與 API

- [ ] HTTP、解析、schema、業務和取消錯誤可區分。
- [ ] 舊請求不能覆蓋新意圖；卸載後不能繼續提交 UI 狀態。
- [ ] 寫入操作的重試和重複提交語義已定義。
- [ ] API 錯誤碼、可否重試和使用者訊息有穩定契約。

### 安全與完整性

- [ ] 所有瀏覽器資料均在伺服器驗證；輸出依語境編碼。
- [ ] 認證與每個物件/動作的授權分開檢查。
- [ ] 唯一、外鍵、非空和跨列規則由資料庫/交易保護。
- [ ] 檔案上傳、cookie、CSRF、XSS、注入和敏感資料最小化已審查。

### 品質與營運

- [ ] 支援瀏覽器、裝置、輸入方式和無障礙標準已定義。
- [ ] 測試覆蓋核心不變量、整合邊界和少量關鍵端到端流程。
- [ ] 已在慢網路、低階裝置、真實資料量、鍵盤和縮放下測試。
- [ ] 有效能基線與預算；依賴和生產產物已審查。
- [ ] 建置可重現，變更可追溯，部署與資料變更可回滾。

## 二十、審閱範圍與資料可信度

- 本次以 MDN 中文學習區入口、核心/擴展模組和課程地圖為範圍，重點交叉閱讀了瀏覽器載入、語義 HTML、CSS 層疊與響應式、事件、非同步、網路、表單、安全、無障礙、效能、測試、框架、伺服器、套件與版本控制等頁面。
- MDN 學習區是持續更新的網站，不是有固定版次的紙本書。本文件是 2026-07-30 的研究快照；具體 API 相容性應再查各 MDN 參考頁的 browser compatibility data。[MDN：Browser compatibility data](https://github.com/mdn/browser-compat-data)
- 課程範例以教學清晰為優先，不等同於完整生產架構。本文中明示為「工程含義」「工程模型」或「建議」的內容，是基於多章和規範交叉推導，不冒充 MDN 原句。
- 中文頁面更新時間不一致；遇到安全、相容性或規範歧義，英文 MDN 最新版與相應標準規範優先。
