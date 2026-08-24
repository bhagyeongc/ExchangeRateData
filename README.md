# FXCOD Exchange Rate Data API

하나은행·KB국민은행의 원화 기준 고시환율과 글로벌 USD 기준 일일 환율을 정적 JSON으로 제공하는 공개 데이터 저장소입니다.

- API 키 없이 HTTP `GET`으로 조회
- 브라우저에서 직접 사용할 수 있도록 CORS 허용
- 최신 환율, 날짜별 스냅샷, 통화별 이력 제공
- 정적 JSON이라 프런트엔드, 서버, 배치 작업에서 간단하게 사용 가능

> 기본 API 주소: **https://exchangerate.fxcod.com**

이 저장소의 데이터는 [FXCOD 환율계산기](https://fxcod.com)에서도 사용합니다.

## 목차

- [빠른 시작](#빠른-시작)
- [API 기본 정보](#api-기본-정보)
- [어떤 파일을 선택해야 하나요?](#어떤-파일을-선택해야-하나요)
- [엔드포인트 한눈에 보기](#엔드포인트-한눈에-보기)
- [응답 구조](#응답-구조)
- [은행 환율 필드](#은행-환율-필드)
- [환율 계산 공식](#환율-계산-공식)
- [사용 예시](#사용-예시)
- [날짜 처리 시 주의사항](#날짜-처리-시-주의사항)
- [오류 처리와 권장 사용 방식](#오류-처리와-권장-사용-방식)
- [저장소 구조](#저장소-구조)
- [데이터 성격과 면책](#데이터-성격과-면책)

## 빠른 시작

최신 하나은행·KB국민은행 통합 환율을 조회합니다.

```bash
curl https://exchangerate.fxcod.com/latest.json
```

JavaScript에서 미국 달러의 하나은행 매매기준율을 가져오는 예시입니다.

```js
const API_BASE = 'https://exchangerate.fxcod.com';

const response = await fetch(`${API_BASE}/latest.json`);
if (!response.ok) {
  throw new Error(`환율 API 오류: ${response.status}`);
}

const data = await response.json();
const usd = data.rates.USD.hana;

console.log(`기준일: ${data.baseDate}`);
console.log(`${usd.unit} ${usd.code} = ${usd.baseRate} KRW`);
```

## API 기본 정보

| 항목 | 내용 |
|---|---|
| Base URL | `https://exchangerate.fxcod.com` |
| 인증 | 필요 없음 |
| 응답 형식 | JSON, UTF-8 |
| 지원 메서드 | `GET`, `HEAD`, `OPTIONS` |
| CORS | `Access-Control-Allow-Origin: *` |
| 은행 데이터 기준 | 대한민국 원화(KRW) |
| 글로벌 데이터 기준 | 미국 달러(USD) |
| 통화 코드 | ISO 4217 형태의 대문자 코드 사용, URL에서는 소문자 권장 |

은행 데이터는 대한민국 영업일 중 여러 차례 갱신되며, 글로벌 기준환율은 하루 한 번 갱신됩니다. 수집 및 배포 상황에 따라 반영 시간이 늦어질 수 있으므로 응답 안의 `baseDate`, `dataDate`, `updatedAt`을 확인하세요.

## 어떤 파일을 선택해야 하나요?

| 만들려는 기능 | 권장 엔드포인트 |
|---|---|
| 최신 하나은행·KB국민은행 환율표 | `/latest.json` |
| 특정 날짜의 두 은행 환율 비교 | `/merged/{YYYY}/{MM}/fx_merged_{YYYY-MM-DD}.json` |
| 특정 은행의 원본에 가까운 정규화 데이터 | `/hana/...` 또는 `/kb/...` |
| 한 통화의 은행 환율 차트 | `/history/{code}/fx_history_365d.json` |
| 장기간 은행 환율 분석 | `/history/{code}/fx_history_all.json` |
| 여러 법정통화의 최신 교차환율 계산 | `/global/latest.json` |
| 글로벌 기준 통화별 KRW 차트 | `/global/history/{code}.json` |

일반적인 환율계산기라면 `/latest.json`, 통화별 차트라면 통화별 이력 파일을 사용하는 구성이 가장 단순합니다.

## 엔드포인트 한눈에 보기

### 최신 은행 환율

| 용도 | 경로 |
|---|---|
| 하나은행·KB국민은행 최신 통합 데이터 | `/latest.json` |
| 최신 데이터의 호환 경로 | `/history/latest.json` |

```text
https://exchangerate.fxcod.com/latest.json
```

### 날짜별 은행 환율

날짜는 `YYYY/MM` 디렉터리와 `YYYY-MM-DD` 파일명으로 구성됩니다.

| 데이터 | 경로 형식 |
|---|---|
| 하나은행 | `/hana/{YYYY}/{MM}/hana_fx_{YYYY-MM-DD}.json` |
| KB국민은행 | `/kb/{YYYY}/{MM}/kb_fx_{YYYY-MM-DD}.json` |
| 두 은행 통합 | `/merged/{YYYY}/{MM}/fx_merged_{YYYY-MM-DD}.json` |

예시:

```text
https://exchangerate.fxcod.com/hana/2026/08/hana_fx_2026-08-24.json
https://exchangerate.fxcod.com/kb/2026/08/kb_fx_2026-08-24.json
https://exchangerate.fxcod.com/merged/2026/08/fx_merged_2026-08-24.json
```

### 통화별 은행 환율 이력

`{code}`에는 `usd`, `jpy`, `eur`처럼 소문자 통화 코드를 사용합니다.

| 범위 | 권장 경로 |
|---|---|
| 최근 365개 수집일 | `/history/{code}/fx_history_365d.json` |
| 보유한 전체 기간 | `/history/{code}/fx_history_all.json` |

예시:

```text
https://exchangerate.fxcod.com/history/usd/fx_history_365d.json
https://exchangerate.fxcod.com/history/jpy/fx_history_all.json
```

`history/fx_history_365d.json`과 `history/fx_history_all.json`에는 여러 통화가 함께 들어 있어 파일이 큽니다. 웹 클라이언트에서는 필요한 통화 하나만 포함하는 위의 통화별 경로를 권장합니다.

### 글로벌 USD 기준환율

| 용도 | 경로 |
|---|---|
| 최신 글로벌 환율 | `/global/latest.json` |
| 날짜별 USD 기준 스냅샷 | `/global/{YYYY}/{MM}/usd_fx_{YYYY-MM-DD}.json` |
| 통화별 KRW 환산 이력 | `/global/history/{code}.json` |

예시:

```text
https://exchangerate.fxcod.com/global/latest.json
https://exchangerate.fxcod.com/global/2026/08/usd_fx_2026-08-23.json
https://exchangerate.fxcod.com/global/history/eur.json
```

## 응답 구조

아래 값은 구조를 설명하기 위한 축약 예시입니다. 실제 최신 숫자와 날짜는 API 응답을 확인하세요.

### `latest.json`

```json
{
  "updatedAt": "2026-08-24T13:41:56.633090+09:00",
  "baseDate": "2026-08-24",
  "prevDate": "2026-08-21",
  "rates": {
    "USD": {
      "hana": {
        "code": "USD",
        "country": "미국",
        "unit": 1,
        "rawTitle": "미국 USD",
        "baseRate": 1380.5,
        "cashBuy": 1404.65,
        "cashSell": 1356.35,
        "send": 1394.0,
        "receive": 1367.0,
        "usConvertRate": 1.0
      },
      "kb": {
        "code": "USD",
        "country": "미국",
        "unit": 1,
        "rawTitle": "미국(달러)",
        "baseRate": 1380.7,
        "cashBuy": 1404.86,
        "cashSell": 1356.54,
        "send": 1394.0,
        "receive": 1367.4,
        "usConvertRate": 1.0
      }
    }
  },
  "prevRates": {}
}
```

`rates`는 `baseDate`의 데이터이고, `prevRates`는 `prevDate`의 데이터입니다. 어떤 통화는 한 은행에서만 제공될 수 있으므로 `hana`와 `kb`의 존재 여부를 각각 확인해야 합니다.

### 날짜별 은행 데이터

하나은행과 KB국민은행 개별 파일은 `meta`와 `rates`로 구성됩니다.

```json
{
  "meta": {
    "baseDate": "2026-08-24",
    "announcedAt": "2026-08-24T13:41:04",
    "queriedAt": "2026-08-24T13:41:50",
    "sequence": 298,
    "requestedDate": "2026-08-24"
  },
  "rates": {
    "EUR": {
      "code": "EUR",
      "country": "유럽연합",
      "unit": 1,
      "rawTitle": "유로 EUR",
      "baseRate": 1612.56,
      "cashBuy": 1644.64,
      "cashSell": 1580.48,
      "send": 1628.68,
      "receive": 1596.44,
      "usConvertRate": 1.1681
    }
  }
}
```

통합 파일은 통화 코드 아래에 은행별 객체를 둡니다.

```json
{
  "meta": {
    "hana": {},
    "kb": {}
  },
  "rates": {
    "EUR": {
      "hana": {},
      "kb": {}
    }
  }
}
```

### 은행 이력 데이터

```json
{
  "code": "USD",
  "updatedAt": "2026-08-24T13:41:56.640089+09:00",
  "totalDays": 365,
  "history": {
    "2026-08-21": {
      "hana": {
        "baseRate": 1390.0
      },
      "kb": {
        "baseRate": 1390.2
      }
    },
    "2026-08-24": {
      "hana": {
        "baseRate": 1380.5
      },
      "kb": {
        "baseRate": 1380.7
      }
    }
  }
}
```

실제 `hana`와 `kb` 객체에는 최신 응답과 동일한 전체 환율 필드가 포함됩니다.

### 글로벌 최신·날짜별 데이터

글로벌 데이터는 USD를 기준으로 합니다. `rates.EUR = 0.85`라면 대략 `1 USD = 0.85 EUR`라는 의미입니다.

```json
{
  "provider": "currency-api",
  "sourceLabel": "글로벌 일일 기준환율",
  "baseCurrency": "USD",
  "dataDate": "2026-08-23",
  "updatedAt": "2026-08-24T12:02:21.724850+09:00",
  "currencies": {
    "USD": "US Dollar",
    "KRW": "South Korean Won",
    "EUR": "Euro"
  },
  "rates": {
    "USD": 1,
    "KRW": 1380.0,
    "EUR": 0.85
  }
}
```

### 글로벌 통화별 KRW 이력

글로벌 이력은 전송량을 줄이기 위해 `[날짜, 환율]` 배열 형태로 압축되어 있습니다. 각 환율은 해당 통화 1단위당 KRW 값입니다.

```json
{
  "code": "EUR",
  "quoteCode": "KRW",
  "startDate": "2025-08-24",
  "endDate": "2026-08-23",
  "totalDays": 364,
  "history": [
    ["2025-08-24", 1623.12194228],
    ["2025-08-25", 1623.30120312]
  ]
}
```

## 은행 환율 필드

| 필드 | 설명 |
|---|---|
| `code` | 통화 코드 |
| `country` | 표시용 국가·지역명 |
| `unit` | 고시 환율이 적용되는 외화 단위. JPY처럼 `100`일 수 있음 |
| `rawTitle` | 원본 고시 화면의 통화명 |
| `baseRate` | 매매기준율 |
| `cashBuy` | 고객이 외화 현찰을 살 때 적용되는 환율 |
| `cashSell` | 고객이 외화 현찰을 팔 때 적용되는 환율 |
| `send` | 해외로 송금할 때 적용되는 환율 |
| `receive` | 해외에서 송금받을 때 적용되는 환율 |
| `usConvertRate` | 원출처가 제공하는 미국 달러 환산율 |

은행 또는 통화에 따라 일부 필드가 `null`이거나 존재하지 않을 수 있습니다. 항상 결측값을 처리하세요.

## 환율 계산 공식

### 은행 데이터: 외화 → 원화

은행 환율은 `unit` 단위 기준입니다.

```text
원화 금액 = 외화 금액 × 선택한 환율 ÷ unit
```

예를 들어 `unit = 100`, `baseRate = 920`인 JPY 데이터로 10,000엔을 계산하면 다음과 같습니다.

```text
10,000 × 920 ÷ 100 = 92,000 KRW
```

원화에서 외화로 역산할 때는 다음 공식을 사용합니다.

```text
외화 금액 = 원화 금액 × unit ÷ 선택한 환율
```

### 글로벌 데이터: 임의 통화 간 교차환율

글로벌 `rates`는 USD 기준이므로 다음 공식으로 두 통화 사이를 변환할 수 있습니다.

```text
변환 결과 = 입력 금액 ÷ rates[입력 통화] × rates[출력 통화]
```

```js
function convertByUsdBase(amount, fromCode, toCode, rates) {
  const fromRate = Number(rates[fromCode]);
  const toRate = Number(rates[toCode]);

  if (!Number.isFinite(fromRate) || fromRate <= 0) {
    throw new Error(`지원하지 않는 입력 통화: ${fromCode}`);
  }
  if (!Number.isFinite(toRate) || toRate <= 0) {
    throw new Error(`지원하지 않는 출력 통화: ${toCode}`);
  }

  return (amount / fromRate) * toRate;
}

const response = await fetch(
  'https://exchangerate.fxcod.com/global/latest.json'
);
if (!response.ok) throw new Error(`HTTP ${response.status}`);

const data = await response.json();
const eurToKrw = convertByUsdBase(100, 'EUR', 'KRW', data.rates);
console.log(`100 EUR = ${eurToKrw.toFixed(2)} KRW`);
```

## 사용 예시

### JavaScript: 은행 선택과 결측값 처리

```js
async function getBankRate(code, preferredBank = 'hana') {
  const response = await fetch(
    'https://exchangerate.fxcod.com/latest.json'
  );
  if (!response.ok) throw new Error(`HTTP ${response.status}`);

  const data = await response.json();
  const currency = data.rates?.[code.toUpperCase()];
  if (!currency) throw new Error(`지원하지 않는 통화: ${code}`);

  const rate = currency[preferredBank]
    ?? currency.hana
    ?? currency.kb;

  if (!rate || !Number.isFinite(Number(rate.baseRate))) {
    throw new Error(`${code}의 유효한 은행 환율이 없습니다.`);
  }

  return {
    baseDate: data.baseDate,
    bank: rate === currency.hana ? 'hana' : 'kb',
    rate
  };
}

const { baseDate, bank, rate } = await getBankRate('JPY', 'kb');
const jpyAmount = 10_000;
const krwAmount = jpyAmount * rate.baseRate / rate.unit;

console.log({ baseDate, bank, krwAmount });
```

### JavaScript: 최근 30개 이력만 차트 데이터로 변환

```js
const response = await fetch(
  'https://exchangerate.fxcod.com/history/eur/fx_history_365d.json'
);
if (!response.ok) throw new Error(`HTTP ${response.status}`);

const data = await response.json();
const chartData = Object.entries(data.history)
  .sort(([dateA], [dateB]) => dateA.localeCompare(dateB))
  .slice(-30)
  .map(([date, banks]) => ({
    date,
    hana: banks.hana?.baseRate ?? null,
    kb: banks.kb?.baseRate ?? null
  }));

console.table(chartData);
```

### JavaScript: 글로벌 압축 이력 읽기

```js
const response = await fetch(
  'https://exchangerate.fxcod.com/global/history/eur.json'
);
if (!response.ok) throw new Error(`HTTP ${response.status}`);

const data = await response.json();
const points = data.history.map(([date, rate]) => ({ date, rate }));

console.log(`${data.code}/${data.quoteCode}`, points.slice(-7));
```

### Python: 최신 USD 환율 조회

```python
import requests

url = "https://exchangerate.fxcod.com/latest.json"
response = requests.get(url, timeout=10)
response.raise_for_status()

data = response.json()
usd = data["rates"]["USD"]

for bank in ("hana", "kb"):
    rate = usd.get(bank)
    if rate:
        print(bank, data["baseDate"], rate["baseRate"])
```

### cURL과 jq

```bash
# 최신 USD 은행별 매매기준율
curl -s https://exchangerate.fxcod.com/latest.json \
  | jq '{baseDate, USD: .rates.USD | {hana: .hana.baseRate, kb: .kb.baseRate}}'

# 글로벌 USD 기준 주요 통화
curl -s https://exchangerate.fxcod.com/global/latest.json \
  | jq '{dataDate, rates: {USD: .rates.USD, KRW: .rates.KRW, EUR: .rates.EUR, JPY: .rates.JPY}}'
```

## 날짜 처리 시 주의사항

URL과 파일명의 날짜는 `requestedDate`, 즉 조회를 요청한 날짜입니다. 주말·공휴일에는 실제 고시 기준일인 `baseDate`가 직전 영업일일 수 있습니다.

예를 들어 다음처럼 파일명은 2025-01-01이지만 실제 기준일은 2024-12-31일 수 있습니다.

```json
{
  "baseDate": "2024-12-31",
  "requestedDate": "2025-01-01"
}
```

따라서 화면 표시, 전일 대비 계산, 시계열 중복 제거에는 파일명만 사용하지 말고 반드시 `meta.baseDate`를 확인하세요.

글로벌 데이터도 원본 데이터에 특정 날짜가 없을 수 있습니다. 이력 배열은 실제 확보된 날짜만 포함하므로 날짜가 매일 연속된다고 가정하지 마세요. `totalDays`는 포함된 데이터 포인트 수로 사용하면 됩니다.

## 오류 처리와 권장 사용 방식

- HTTP 상태가 `200`인지 확인한 후 JSON을 파싱하세요.
- 아직 수집되지 않았거나 제공되지 않는 날짜는 `404`가 될 수 있습니다.
- 네트워크 오류에는 짧은 지수 백오프 재시도를 권장합니다.
- 동일 데이터를 반복 요청하지 말고 브라우저·서버 캐시를 활용하세요.
- `rates[code]`, `hana`, `kb` 및 개별 환율 필드가 없을 가능성을 처리하세요.
- 최신성이 중요한 경우 `baseDate`, `dataDate`, `updatedAt`을 검증하세요.
- 대용량 전체 이력보다 통화별 이력 파일을 우선 사용하세요.

```js
async function fetchJsonWithRetry(url, maxAttempts = 3) {
  let lastError;

  for (let attempt = 1; attempt <= maxAttempts; attempt += 1) {
    try {
      const response = await fetch(url);
      if (!response.ok) throw new Error(`HTTP ${response.status}`);
      return await response.json();
    } catch (error) {
      lastError = error;
      if (attempt < maxAttempts) {
        await new Promise(resolve =>
          setTimeout(resolve, 500 * (2 ** (attempt - 1)))
        );
      }
    }
  }

  throw lastError;
}
```

## 저장소 구조

```text
.
├── latest.json
├── hana/
│   └── YYYY/MM/hana_fx_YYYY-MM-DD.json
├── kb/
│   └── YYYY/MM/kb_fx_YYYY-MM-DD.json
├── merged/
│   └── YYYY/MM/fx_merged_YYYY-MM-DD.json
├── history/
│   ├── latest.json
│   ├── {code}/
│   │   ├── fx_history_365d.json
│   │   └── fx_history_all.json
│   ├── fx_history_365d.json
│   └── fx_history_all.json
├── global/
│   ├── latest.json
│   ├── YYYY/MM/usd_fx_YYYY-MM-DD.json
│   └── history/{code}.json
├── CNAME
└── _headers
```

## 데이터 성격과 면책

- 본 저장소는 하나은행 또는 KB국민은행의 공식 API나 공식 저장소가 아닙니다.
- 은행별 환율은 각 은행이 공개한 고시환율을 수집·정규화한 데이터입니다.
- 글로벌 데이터는 일일 기준환율이며 은행의 현찰·송금 환율을 포함하지 않습니다.
- 실제 환전·송금 시에는 거래 시점, 고시 회차, 우대율, 수수료, 영업점 및 서비스 조건에 따라 적용 금액이 달라질 수 있습니다.
- 이 데이터는 참고용이며 금융 거래, 회계, 세무, 법률 판단의 최종 근거로 사용하기 전에 공식 채널에서 다시 확인해야 합니다.
- 서비스의 무중단 제공, 특정 갱신 시각, 모든 날짜·통화의 완전성을 보장하지 않습니다.

오류나 누락을 발견했다면 이 저장소의 [Issues](https://github.com/bhagyeongc/ExchangeRateData/issues)에 재현 가능한 URL, 통화 코드, 날짜와 함께 알려주세요.
