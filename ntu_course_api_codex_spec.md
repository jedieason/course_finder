# 臺大課程網：特定時段搜尋與推估錄取率排序技術規格

## 1. 適用範圍與重要聲明

- 本規格是 2026-08-19 由臺大課程網 v3.4.0 前端實際使用方式還原而來。
- 這些是網站的公開、但未正式文件化的內部 API；路徑、欄位或限制未來可能改變。
- 實測不需要登入 Cookie 或 Token，但程式仍應遵守網站條款、限制併發並加上重試退避。
- 「錄取率」是根據當下 `remaining` 與 `registered` 推估，不是校方保證的正式錄取率。
- 真實錄取仍可能受到加選方式、年級/系所限制、預配、保留名額及其他優先規則影響。

## 2. API 基底與端點

API 基底：

```text
https://course.ntu.edu.tw/api
```

### 2.1 搜尋課程

```http
POST /api/v1/courses/search/quick?lang=zh-TW
Accept: application/json
Content-Type: application/json
```

完整 URL：

```text
https://course.ntu.edu.tw/api/v1/courses/search/quick?lang=zh-TW
```

請求本文：

```json
{
  "query": {
    "keyword": "",
    "time": [[], [], [], [], [], []],
    "timeStrictMatch": true,
    "isFullYear": null,
    "excludedKeywords": [],
    "enrollMethods": [],
    "isEnglishTaught": false,
    "isDistanceLearning": false,
    "hasChanged": false,
    "isAdditionalCourse": false,
    "noPrerequisite": false,
    "isCanceled": false,
    "isIntensive": false,
    "isPrecise": true,
    "type": "quick",
    "departments": [],
    "isCompulsory": null
  },
  "batchSize": 100,
  "pageIndex": 0,
  "sorting": "correlation"
}
```

實測限制及行為：

- `batchSize` 最大為 `100`；送 `1256` 會收到驗證錯誤。
- `pageIndex` 從 `0` 開始。
- 回應頂層至少包含 `totalCount` 與 `courses`。
- 先抓第 0 頁，再以 `Math.ceil(totalCount / batchSize)` 算總頁數。
- `sorting` 本次只驗證過 `"correlation"`，不要猜測其他值。
- 如需鎖定特定學期，可嘗試在 `query` 加入 `semester: "115-1"`；本次頁面未明示傳送此欄，API 自動回傳當期 `115-1`。程式應以回傳課程的 `semester` 再次驗證。

搜尋回應摘要：

```json
{
  "totalCount": 1256,
  "courses": [
    {
      "id": "b42fe9df-b79b-4076-93ab-ccf38a1db095",
      "serial": "58362",
      "identifier": "AC2004",
      "code": "603 21200",
      "name": "土壤學實驗",
      "teacher": {
        "id": "603077",
        "name": "許正一"
      },
      "class": "01",
      "schedules": [
        {
          "weekday": 2,
          "intervals": ["7", "8", "9"],
          "classroom": { "name": "農化二211" }
        }
      ],
      "credits": 1,
      "totalStudentQuota": 30,
      "otherDepartmentsStudentQuota": 5,
      "enrollMethod": 2,
      "status": "Normal",
      "semester": "115-1",
      "hostDepartment": "農業化學系"
    }
  ]
}
```

其中 `course.id`（UUID）是查詢即時登記狀態的必要鍵；不要用流水號代替。

### 2.2 取得單門課的即時登記狀態

```http
GET /api/v1/courses/enroll-status/{courseId}
Accept: application/json
```

範例：

```text
https://course.ntu.edu.tw/api/v1/courses/enroll-status/b42fe9df-b79b-4076-93ab-ccf38a1db095
```

回應：

```json
{
  "enrolled": 18,
  "enrolledOtherDept": 0,
  "registered": 1,
  "remaining": 12
}
```

欄位含義：

- `enrolled`：已選上人數。
- `enrolledOtherDept`：外系已選上人數。
- `registered`：目前已登記、正在競爭名額的人數。
- `remaining`：網站後端計算的剩餘可錄取名額。

本次沒有發現批次查詢登記狀態的端點，因此需對每個 `course.id` 各發一個 GET。

建議：

- 併發數控制在 10–20；本次使用 20。
- 每次請求最多重試 5 次，採漸增退避，例如 400、800、1200、1600 毫秒。
- 若追求最新結果，不要永久快取狀態；可不快取，或使用很短的 TTL。
- 課程中繼資料可以快取，但 `registered` 和 `remaining` 會快速變動。

## 3. 特定時段的編碼

### 3.1 API 的 `query.time`

`query.time` 必須是**剛好 6 個陣列**：

```js
const time = [
  [], // 1 = 星期一
  [], // 2 = 星期二
  [], // 3 = 星期三
  [], // 4 = 星期四
  [], // 5 = 星期五
  [], // 6 = 星期六
];
```

每個內層陣列放節次字串。前端接受的節次為：

```text
0, 1, 2, 3, 4, 5, 6, 7, 8, 9, X, A, B, C, D
```

例如：

```js
// 星期二 6、7、8、9；星期三 1、2
const time = [
  [],
  ["6", "7", "8", "9"],
  ["1", "2"],
  [],
  [],
  [],
];
```

建立時段矩陣：

```js
function buildTimeMatrix(slots) {
  const result = Array.from({ length: 6 }, () => []);
  for (const { weekday, interval } of slots) {
    if (!Number.isInteger(weekday) || weekday < 1 || weekday > 6) {
      throw new Error(`weekday 必須是 1–6：${weekday}`);
    }
    result[weekday - 1].push(String(interval));
  }
  for (const day of result) {
    day.sort((a, b) => String(a).localeCompare(String(b), "en", { numeric: true }));
  }
  return result;
}
```

### 3.2 網頁 URL 的 `t` 參數

網頁 URL 使用「星期數字 + 節次」的扁平編碼：

- `26` = 星期二第 6 節。
- `2A` = 星期二 A 節。
- `34` = 星期三第 4 節。
- `5X` = 星期五 X 節。

例如：

```text
t=26,27,28,29,31,32
```

前端會把它轉成：

```json
[
  [],
  ["6", "7", "8", "9"],
  ["1", "2"],
  [],
  [],
  []
]
```

相關 URL 參數：

- `t`：時段代碼，以逗號分隔。
- `tsm=true`：嚴格時段比對，對應 API 的 `timeStrictMatch: true`。
- `ek=詞1,詞2`：排除關鍵字，對應 `excludedKeywords`。

本次頁面的 28 個時段：

```text
26,27,28,29,2A,2B,2C,2D,2X,
30,31,32,33,34,
48,49,4A,4B,4C,4D,4X,
58,59,5A,5B,5C,5D,5X
```

對應 API：

```json
[
  [],
  ["6", "7", "8", "9", "A", "B", "C", "D", "X"],
  ["0", "1", "2", "3", "4"],
  ["8", "9", "A", "B", "C", "D", "X"],
  ["8", "9", "A", "B", "C", "D", "X"],
  []
]
```

## 4. 搜尋查詢欄位

快速搜尋時使用的完整查詢模型：

| 欄位 | 型別 | 本次值／用途 |
|---|---|---|
| `keyword` | string | 課名、教師或流水號關鍵字；空字串代表不限 |
| `time` | string[][] | 6 天的節次矩陣 |
| `timeStrictMatch` | boolean | 本次為 `true`，頁面顯示「嚴格」 |
| `isFullYear` | boolean \| null | 本次為 `null` |
| `excludedKeywords` | string[] | 排除課名關鍵字 |
| `enrollMethods` | number[] | 加選方式，可用 1–3；空陣列代表不限 |
| `isEnglishTaught` | boolean | 是否只要英文授課 |
| `isDistanceLearning` | boolean | 是否只要遠距課程 |
| `hasChanged` | boolean | 是否只要有異動課程 |
| `isAdditionalCourse` | boolean | 是否只要加開課程 |
| `noPrerequisite` | boolean | 是否只要無先修限制 |
| `isCanceled` | boolean | 是否只要停開課程 |
| `isIntensive` | boolean | 是否只要密集課程 |
| `isPrecise` | boolean | 無 URL 參數時前端預設為 `true` |
| `type` | string | 快速搜尋固定為 `"quick"` |
| `departments` | array | 快速搜尋使用空陣列 |
| `isCompulsory` | boolean \| null | 快速搜尋使用 `null` |
| `semester` | string（選填） | 例如 `"115-1"`；回應後仍需核對 |

本次實際查詢另使用：

```json
"excludedKeywords": ["一上", "三上", "二上", "國文", "大學英文", "專題"]
```

## 5. 錄取率計算

### 5.1 本次實際使用的公式

```js
function estimateAdmissionRate({ remaining, registered }) {
  remaining = Number(remaining);
  registered = Number(registered);

  if (!Number.isFinite(registered) || registered <= 0) return null;
  if (!Number.isFinite(remaining)) return null;

  return Math.min(1, Math.max(0, remaining / registered));
}
```

百分比：

```js
const rate = estimateAdmissionRate(status);
const ratePct = rate === null ? null : rate * 100;
```

規則：

- `registered > 0` 且 `remaining = 0`：錄取率為 0%。
- `remaining >= registered > 0`：上限封頂為 100%。
- `registered = 0`：回傳 `null`，不能硬算成 0%，也不應進入由低到高排名。
- 排序時使用未四捨五入的 `rate`；只在顯示時四捨五入。

### 5.2 為何使用 `remaining / registered`

- `remaining` 是目前仍可分配的席次。
- `registered` 是目前正在競爭這些席次的人數。
- `enrolled` 是已經選上的人，不是目前這一輪的競爭母體。
- 因此不應用 `enrolled / registered`，也不應用 `totalStudentQuota / registered`。
- 不要自行用 `totalStudentQuota - enrolled` 重算剩餘名額；預配、外系名額、超收或後端規則可能讓結果與官方 `remaining` 不同。

## 6. 排序規則

本次實際使用：

```js
const ranked = rows
  .filter((course) => course.estimatedAdmissionRate !== null)
  .sort((a, b) =>
    a.estimatedAdmissionRate - b.estimatedAdmissionRate ||
    b.enrollmentStatus.registered - a.enrollmentStatus.registered ||
    a.enrollmentStatus.remaining - b.enrollmentStatus.remaining ||
    a.name.localeCompare(b.name, "zh-Hant")
  );
```

順序為：

1. 推估錄取率由低到高。
2. 錄取率相同時，`registered` 由高到低。
3. 再相同時，`remaining` 由低到高。
4. 再相同時，中文課名升冪。

如需完全穩定的結果，建議最後再加：

```js
|| a.serial.localeCompare(b.serial)
```

## 7. 完整抓取流程

1. 接收使用者指定的星期、節次、排除關鍵字和其他篩選。
2. 將星期/節次轉成長度固定為 6 的 `query.time`。
3. POST 搜尋 API，第 0 頁使用 `batchSize: 100`。
4. 讀取 `totalCount`，計算總頁數。
5. 抓取剩餘頁面；搜尋頁建議併發 4。
6. 合併所有 `courses`。
7. 驗證：
   - `courses.length === totalCount`
   - `new Set(courses.map(c => c.id)).size === totalCount`
8. 對每個 `course.id` GET 登記狀態；建議併發 10–20。
9. 將狀態合併至課程資料。
10. 計算未四捨五入的錄取率。
11. 排除 `registered = 0` 的無法估計課程。
12. 依第 6 節排序，輸出全部或前 N 名。
13. 在結果中附上資料抓取時間、查詢條件與公式。

## 8. Node.js 18+ 核心範例

```js
const API = "https://course.ntu.edu.tw/api";

async function fetchJson(url, options = {}, retries = 5) {
  let lastError;
  for (let attempt = 1; attempt <= retries; attempt++) {
    try {
      const response = await fetch(url, {
        ...options,
        headers: {
          accept: "application/json",
          ...(options.body ? { "content-type": "application/json" } : {}),
          ...(options.headers ?? {}),
        },
      });
      const text = await response.text();
      if (!response.ok) throw new Error(`HTTP ${response.status}: ${text.slice(0, 300)}`);
      return JSON.parse(text);
    } catch (error) {
      lastError = error;
      if (attempt < retries) {
        await new Promise(resolve => setTimeout(resolve, 400 * attempt));
      }
    }
  }
  throw lastError;
}

async function mapConcurrent(items, concurrency, mapper) {
  const output = new Array(items.length);
  let cursor = 0;
  const workers = Array.from({ length: concurrency }, async () => {
    while (true) {
      const index = cursor++;
      if (index >= items.length) return;
      output[index] = await mapper(items[index], index);
    }
  });
  await Promise.all(workers);
  return output;
}

function buildTimeMatrix(slots) {
  const result = Array.from({ length: 6 }, () => []);
  for (const { weekday, interval } of slots) {
    if (!Number.isInteger(weekday) || weekday < 1 || weekday > 6) {
      throw new Error(`非法 weekday: ${weekday}`);
    }
    result[weekday - 1].push(String(interval));
  }
  return result;
}

function estimateAdmissionRate(status) {
  const remaining = Number(status.remaining);
  const registered = Number(status.registered);
  if (!Number.isFinite(remaining) || !Number.isFinite(registered) || registered <= 0) {
    return null;
  }
  return Math.min(1, Math.max(0, remaining / registered));
}

async function searchAndRank({ slots, excludedKeywords = [], keyword = "", limit = 30 }) {
  const query = {
    keyword,
    time: buildTimeMatrix(slots),
    timeStrictMatch: true,
    isFullYear: null,
    excludedKeywords,
    enrollMethods: [],
    isEnglishTaught: false,
    isDistanceLearning: false,
    hasChanged: false,
    isAdditionalCourse: false,
    noPrerequisite: false,
    isCanceled: false,
    isIntensive: false,
    isPrecise: true,
    type: "quick",
    departments: [],
    isCompulsory: null,
  };

  const batchSize = 100;
  const searchUrl = `${API}/v1/courses/search/quick?lang=zh-TW`;
  const fetchPage = pageIndex => fetchJson(searchUrl, {
    method: "POST",
    body: JSON.stringify({ query, batchSize, pageIndex, sorting: "correlation" }),
  });

  const firstPage = await fetchPage(0);
  const pageCount = Math.ceil(firstPage.totalCount / batchSize);
  const otherPages = await mapConcurrent(
    Array.from({ length: pageCount - 1 }, (_, i) => i + 1),
    4,
    fetchPage,
  );

  const courses = [firstPage, ...otherPages].flatMap(page => page.courses);
  if (courses.length !== firstPage.totalCount) {
    throw new Error(`筆數不符：${courses.length}/${firstPage.totalCount}`);
  }
  if (new Set(courses.map(course => course.id)).size !== courses.length) {
    throw new Error("課程 UUID 重複");
  }

  const statuses = await mapConcurrent(courses, 15, course =>
    fetchJson(`${API}/v1/courses/enroll-status/${course.id}`),
  );

  const ranked = courses
    .map((course, index) => {
      const enrollmentStatus = statuses[index];
      return {
        ...course,
        enrollmentStatus,
        estimatedAdmissionRate: estimateAdmissionRate(enrollmentStatus),
      };
    })
    .filter(course => course.estimatedAdmissionRate !== null)
    .sort((a, b) =>
      a.estimatedAdmissionRate - b.estimatedAdmissionRate ||
      b.enrollmentStatus.registered - a.enrollmentStatus.registered ||
      a.enrollmentStatus.remaining - b.enrollmentStatus.remaining ||
      a.name.localeCompare(b.name, "zh-Hant") ||
      a.serial.localeCompare(b.serial)
    )
    .map((course, index) => ({
      rank: index + 1,
      ratePct: Number((course.estimatedAdmissionRate * 100).toFixed(4)),
      ...course,
    }));

  return {
    collectedAt: new Date().toISOString(),
    totalCount: firstPage.totalCount,
    rankableCount: ranked.length,
    query,
    courses: limit == null ? ranked : ranked.slice(0, limit),
  };
}

// 範例：星期二 6–9、星期三 1–2，排除「國文」與「專題」。
const result = await searchAndRank({
  slots: [
    { weekday: 2, interval: "6" },
    { weekday: 2, interval: "7" },
    { weekday: 2, interval: "8" },
    { weekday: 2, interval: "9" },
    { weekday: 3, interval: "1" },
    { weekday: 3, interval: "2" },
  ],
  excludedKeywords: ["國文", "專題"],
  limit: 30,
});

console.log(JSON.stringify(result, null, 2));
```

## 9. 可直接餵給 Codex 的任務提示

```text
請用 Node.js 18+ 實作一個臺大課程搜尋與錄取率排序工具。

要求：
1. 接受星期 1–6 與節次 0–9/X/A–D，轉成 6 個陣列的 query.time。
2. POST https://course.ntu.edu.tw/api/v1/courses/search/quick?lang=zh-TW。
3. batchSize 最大 100，pageIndex 從 0 開始；先讀 totalCount，再抓齊所有分頁。
4. 搜尋請求 body 使用本規格列出的完整 query 欄位，sorting 使用 correlation。
5. 驗證回傳筆數等於 totalCount，且 course.id UUID 全部唯一。
6. 對每個 UUID GET https://course.ntu.edu.tw/api/v1/courses/enroll-status/{courseId}。
7. 登記狀態請求使用 10–20 併發、最多 5 次重試與退避；不得把舊快取當成最新資料。
8. 推估錄取率 = registered > 0 ? min(1, max(0, remaining / registered)) : null。
9. 不可用 enrolled/registered、totalStudentQuota/registered，亦不可自行倒算 remaining。
10. 排除錄取率為 null 的課程；依未四捨五入錄取率升冪，再依 registered 降冪、remaining 升冪、中文課名升冪、流水號升冪排序。
11. 輸出抓取時間、查詢條件、總筆數、可排名筆數，以及全部排名或前 N 名。
12. 結果至少包含：排名、錄取率%、剩餘名額、已登記、已選上、流水號、課號、課名、教師、星期節次、學期、課程網址。
13. 明確標註這是即時推估，不是校方正式錄取率；內部 API 可能變更。
```

## 10. 本次完整性驗證結果

- 搜尋 API 回報：1,256 筆。
- 實際取得課程：1,256 筆。
- 唯一課程 UUID：1,256 個。
- 登記狀態：1,256 筆。
- `registered > 0`、可計算錄取率：1,013 筆。
- `registered = 0`、不可計算：243 筆。

