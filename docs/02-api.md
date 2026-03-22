## API 명세

> 1. 회차 구매 API

1-1. endpoint
```text
POST /users/{userId}/episodes/{episodeId}/purchase
```

1-2. request
```text
path variable: 
    userId 사용자 ID
    episodeId 회차 ID
body: 
    없음
```

1-3. response
```text
200 OK
{
  "status": 200,
  "message": "구매가 완료되었습니다.",
  "data": {
    "purchaseId",
    "episode": {
      "episodeId",
      "title",
      "price"
    },
    "deductedCoin",
    "purchasedAt"
  }
}
```
```text
error
{
"status": 404,
"message": "존재하지 않는 사용자입니다.",
"data": null
}
...
```
| HTTP Status | 설명 | message |
|---|---|---|
| 404 | `USER_NOT_FOUND` | 존재하지 않는 사용자입니다. |
| 404 | `EPISODE_NOT_FOUND` | 존재하지 않는 회차입니다. |
| 409 | `ALREADY_PURCHASED` | 이미 구매한 회차입니다. |
| 409 | `ALREADY_PURCHASED` | 동시 요청으로 인한 중복 구매가 감지되었습니다. |
| 400 | `INVALID_COIN_AMOUNT` | 코인 금액이 올바르지 않습니다. (0원, 음수, 숫자 외 형식 불가) |
| 400 | `MISSING_REQUIRED_FIELD` | 필수 요청값이 누락되었습니다. |
| 500 | `PURCHASE_FAILED` | 결제 처리 중 오류가 발생했습니다. |


> 2. 회차 열람 API

2-1. endpoint
```text
GET /users/{userId}/episodes/{episodeId}/access
```

2-2. request
```text
path variable: 
    userId 사용자 ID
    episodeId 회차 ID
body: 
    없음
```

2-3. response
```text
200 OK
{
  "status": 200,
  "message": "열람 권한이 확인되었습니다.",
    "data": {
      "episode": {
        "episodeId",
        "title",
        "episodeNumber"
      },
      "content": {
        "contentUrl",
        "grantedAt"
      }
  }
}
```
```text
error
{
  "status": 403,
  "message": "열람 권한이 없습니다.",
  "data": null
}
...
```
| HTTP Status | 설명 | message |
|-------------|---|---|
| 404         | `USER_NOT_FOUND` | 존재하지 않는 사용자입니다. |
| 404         | `EPISODE_NOT_FOUND` | 존재하지 않는 회차입니다. |
| 403         | `ACCESS_DENIED` | 열람 권한이 없습니다. |
