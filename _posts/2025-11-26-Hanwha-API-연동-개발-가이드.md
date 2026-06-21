---
layout: post
title: 한화리조트 PMS API 연동 개발 가이드
date: 2025-11-26
sitemap: true
hide_last_modified: true
categories:
- dev
- allmytour
tags:
- hanwha
- resort
- pms
- api
- spring
- webclient
---

* toc
{:toc .large-only}

# 한화리조트 PMS API 연동 개발 가이드

올마이투어와 한화리조트 PMS(iGate LCB) 간의 외부 API 연동 개발 내용을 정리한 문서입니다.  
Spring Boot + WebClient(Reactor) 기반으로 구현되었으며, 모든 API는 JSON 형식으로 통신합니다.

---

## 시스템 개요

| 항목 | 내용 |
|------|------|
| 연동 대상 | 한화리조트 PMS (iGate LCB 게이트웨이) |
| 통신 방식 | HTTPS POST (단일 엔드포인트, serviceId로 API 구분) |
| 데이터 형식 | JSON |
| 인증 방식 | 요청 Body 내 `CUST_NO`, `CONT_NO` 포함 + 헤더 `Uuid` (UUID v4) |
| HTTP Client | Spring WebClient (Project Reactor) |
| 타임아웃 | 60초 |
| 스케줄러 | 재고 동기화 매일 01:00, 요금 동기화 매일 05:00 |

> 연동 API는 단일 엔드포인트에 대해 요청 Body의 `SystemHeader.INTFC_ID` 값으로 서비스를 구분합니다.

---

## 서비스 인터페이스 목록

| 서비스명 | INTFC_ID | SERVICE_ID | 설명 |
|----------|----------|------------|------|
| PACKAGELIST | `LCB00HBSREMPRR9906` | `HBSREMPRR9906` | 패키지 목록 조회 |
| PACKAGEDETAIL | `LCB00HBSREMPRR9931` | `HBSREMPRR9931` | 패키지 구성(상세) 조회 |
| BLOCK | `LCB00HBSREMPRR9905` | `HBSREMPRR9905` | 케파(가용 객실 재고) 조회 |
| RATE | `LCB00HBSREMPRR9907` | `HBSREMPRR9907` | 일별 객실 요금 조회 |
| RESERVATION | `LCB00HBSREMPRR9901` | `HBSREMPRR9901` | 예약 생성 |
| CANCEL | `LCB00HBSREMPRR9902` | `HBSREMPRR9902` | 예약 취소 |
| RETRIEVE | `LCB00HBSREMPRR9903` | `HBSREMPRR9903` | 예약 조회 |
| MODIFY | `LCB00HBSREMPRR9904` | `HBSREMPRR9904` | 예약 수정 (예비) |

---

## 공통 요청/응답 구조

### 요청 (CommonRequest)

모든 한화 API 요청은 아래 공통 헤더 구조를 사용합니다.

```json
{
  "systemHeader": {
    "INTFC_ID": "LCB00HBSREMPRR9906",
    "SERVICE_ID": "HBSREMPRR9906"
  },
  "transactionHeader": {
    "TRANS_DT": "20251126",
    "TRANS_TM": "120000",
    "TRANS_SEQ_NO": "..."
  },
  "messageHeader": {
    "MSG_PRCS_RSLT_CD": null,
    "MSG_DATA_SUB_RPTT_CNT": null,
    "MSG_ETC": null
  },
  "data": { ... }
}
```

### 응답 성공 판별

```java
// messageHeader.MSG_PRCS_RSLT_CD == "0"  AND
// messageHeader.MSG_DATA_SUB[0].MSG_CD == "SCMI000001"
```

- `MSG_PRCS_RSLT_CD`가 `"0"`이고 `MSG_CD`가 `"SCMI000001"`이면 정상 처리
- 그 외는 `HanwhaReservationException` 발생

### HTTP 요청 헤더

```
Content-Type: application/json
Uuid: {UUID v4 랜덤 생성}
```

---

## API 목록 (올마이투어 서버)

### Contents Controller (`/v1/contents`)

| Method | Endpoint | 설명 | 호출 방향 |
|--------|----------|------|-----------|
| POST | `/v1/contents/sync-package` | 패키지 동기화 | 내부 호출 → 한화 PMS |
| GET | `/v1/contents/search-package` | 패키지 목록 조회 (DB) | 내부 조회 |
| POST | `/v1/contents/sync-block` | 케파(재고) 동기화 | 내부 호출 → 한화 PMS |
| POST | `/v1/contents/sync-rate` | 일별 객실료 동기화 | 내부 호출 → 한화 PMS |

### Realtime Controller (`/v1/realtime`)

| Method | Endpoint | 설명 | 호출 방향 |
|--------|----------|------|-----------|
| POST | `/v1/realtime/reservation` | 예약 생성 | 올마이투어 → 한화 PMS |
| PUT | `/v1/realtime/reservation` | 예약 취소 | 올마이투어 → 한화 PMS |
| GET | `/v1/realtime/reservation/{orderNumber}` | 예약 조회 | 올마이투어 → 한화 PMS |
| POST | `/v1/realtime/reservation/{orderNumber}` | 예약 복구 | 올마이투어 → 한화 PMS |

---

## API별 처리 프로세스

---

### 1. 패키지 동기화 `POST /v1/contents/sync-package`

한화 PMS에서 영업장별 패키지 목록 및 구성 정보를 조회하여 DB에 동기화합니다.

**처리 흐름**

```
POST /v1/contents/sync-package
  ↓
[Step 1] 패키지 목록 조회 (PACKAGELIST - HBSREMPRR9906)
  t_product_mapping에서 한화 연동 숙소(LOC_CD) 목록 조회
  → 한화 PMS API 호출: 향후 12개월치 월별 1일 기준으로 패키지 목록 요청
  → PackageListResponse 수신 (패키지번호/명/유효기간/판매기간/연박수)
  ↓
[Step 2] 패키지 구성 조회 (PACKAGEDETAIL - HBSREMPRR9931)
  패키지 번호 목록 추출 (distinct)
  → Flux 병렬 처리 (Schedulers.boundedElastic)로 패키지 상세 일괄 조회
  → PackageDetailResponse 수신 (영업장코드/룸타입코드/객실명)
  ↓
[Step 3] DB 저장
  기 등록된 hanwha_room 정보와 매핑
  → (영업장코드 + 룸타입코드) 기준으로 product_idx, room_type_idx 매핑
  → HanwhaRatePlan 생성 후 DB upsert (hanwha_rate_plan 테이블)
```

**요청 파라미터**

| 필드 | 설명 |
|------|------|
| `productIdxList` | 동기화 대상 상품 Idx 목록 (비어있으면 전체) |

**한화 패키지 목록 요청 주요 필드 (ds_search)**

| 필드 | 설명 |
|------|------|
| `CUST_NO` | 고객 번호 (설정값) |
| `CONT_NO` | 계약 번호 (설정값) |
| `LOC_CD` | 영업장 코드 |
| `ARRV_DATE` | 도착일 (yyyyMMdd, 월 1일 기준 12개월) |
| `PAKG_NO` | 패키지 번호 (전체 조회 시 null) |

---

### 2. 패키지 목록 조회 `GET /v1/contents/search-package`

DB에 저장된 한화 패키지 목록을 조회하는 내부 API입니다. (외부 API 미호출)

**처리 흐름**

```
GET /v1/contents/search-package
  → ContentsService.searchPackage(request)
  → DB 조회 (hanwha_rate_plan 테이블)
  → List<PackageSearchList> 반환
```

---

### 3. 케파(재고) 동기화 `POST /v1/contents/sync-block`

한화 PMS에서 일별 객실 가용 재고를 조회하여 DB에 업데이트합니다.  
**스케줄러에 의해 매일 01:00 자동 실행됩니다.**

**처리 흐름**

```
POST /v1/contents/sync-block  (또는 스케줄러 자동 실행)
  ↓
매핑 정보 조회
  → hanwha_mapping 테이블에서 대상 목록 조회
  → 룸온리 상품 / 패키지 상품으로 분류
  ↓
[룸온리 상품] 재고 업데이트
  → 케파 API 호출 (BLOCK - HBSREMPRR9905)
  → 0.5초 간격 직렬 처리 (한화 API 병렬 처리 시 오류 방지)
  → KepaResponse.DsRoomStatus (세션날짜, 예약가능수) 수신
  → product_room_stock 테이블 upsert (avail, stock, date)
  ↓
[패키지 상품] 요금제 가용 업데이트
  → 케파 API 호출 (BLOCK - HBSREMPRR9905, 패키지번호 포함)
  → product_rateplan_price_avail 테이블 upsert (avail, date)
```

**케파 요청 주요 필드 (KepaRequest)**

| 필드 | 설명 |
|------|------|
| `CUST_NO` | 고객 번호 |
| `CONT_NO` | 계약 번호 |
| `LOC_CD` | 영업장 코드 |
| `ROOM_TYPE_CD` | 룸타입 코드 |
| `PAKG_NO` | 패키지 번호 (룸온리는 null) |

**케파 응답 주요 필드 (DsRoomStatus)**

| 필드 | 설명 |
|------|------|
| `LOC_CD` | 영업장 코드 |
| `ROOM_TYPE_CD` | 룸타입 코드 |
| `SESN_DATE` | 세션 날짜 (yyyyMMdd) |
| `RSRV_POSBL_CNT` | 예약 가능 수량 |

**요청 파라미터**

| 필드 | 설명 |
|------|------|
| `allSync` | 전체 동기화 여부 (true: 전체) |
| `productIdxList` | 특정 상품 Idx 목록 |

---

### 4. 일별 객실료 동기화 `POST /v1/contents/sync-rate`

한화 PMS에서 일별 객실 요금을 조회하여 DB에 업데이트합니다.  
**스케줄러에 의해 매일 05:00 자동 실행됩니다.**

**처리 흐름**

```
POST /v1/contents/sync-rate  (또는 스케줄러 자동 실행)
  ↓
매핑 정보 조회
  → hanwha_mapping 테이블에서 대상 목록 조회
  → 룸온리 상품 / 패키지 상품으로 분류
  ↓
[룸온리 상품] 요금 업데이트
  오늘 기준 2개월치, 1일 간격으로 요청 목록 생성
  → 요금 API 호출 (RATE - HBSREMPRR9907)
  → 0.5초 간격 직렬 처리
  → RateResponse.DsResult (영업일, 객실요금, 수수료율 등) 수신
  → 마진율 계산: otaPrice → duePrice, netPrice, price 순차 계산
  → product_rateplan_price 테이블 upsert
  ↓
[패키지 상품] 요금 업데이트
  패키지 유효기간(minDate ~ maxDate), ovntCnt 간격으로 요청 목록 생성
  → 요금 API 호출 (RATE - HBSREMPRR9907)
  → 동일 방식으로 요금 계산 및 DB upsert
```

**요금 API 요청 주요 필드 (RateRequest)**

| 필드 | 설명 |
|------|------|
| `LOC_CD` | 영업장 코드 |
| `ROOM_TYPE_CD` | 룸타입 코드 |
| `PAKG_NO` | 패키지 번호 (룸온리는 null) |
| `ARRV_DATE` | 도착일 (yyyyMMdd) |
| `OVNT_CNT` | 연박 수 (룸온리: 1, 패키지: ovntCnt) |

**요금 응답 주요 필드 (DsResult)**

| 필드 | 설명 |
|------|------|
| `BSN_DATE` | 영업일 (yyyyMMdd) |
| `ROOM_RATE` | 객실 요금 |
| `ORIG_ROOM_RATE` | 원래 객실 요금 |
| `CALC_ROOM_RATE` | 계산된 객실 요금 (마진 계산 기준) |
| `FEE_RATE` | 수수료율 |
| `UPGRD_YN` | 업그레이드 여부 |

**마진 계산 로직**

```java
// otaPrice = CALC_ROOM_RATE
// duePrice = otaPrice × inDuePriceRate  (올림 없음)
// netPrice = otaPrice × inNetPriceRate
// price    = netPrice × inPriceRate
// → product_rateplan_price 테이블에 저장
```

---

### 5. 예약 생성 `POST /v1/realtime/reservation`

올마이투어에서 한화 PMS로 실시간 예약을 전송합니다.

**처리 흐름**

```
올마이투어 결제 완료
  → POST /v1/realtime/reservation (JSON)
  ↓
[1] 예약 요청 로그 저장 (reservation_log)
  ↓
[2] 한화 예약 사전 저장 (hanwha_reservation)
  rateplanIdx 기반으로 hanwha_room_info 조회
  → 영업장코드(LOC_CD), 룸타입코드, 패키지번호 확인
  ↓
[3] 한화 PMS 예약 API 호출 (RESERVATION - HBSREMPRR9901)
  WebClient POST → 한화 이게이트 단일 엔드포인트
  ↓
[4] 응답 검증
  MSG_PRCS_RSLT_CD == "0" AND MSG_CD == "SCMI000001"
  → 성공: hanwha_reservation 업데이트 (bookingStatus=COMPLETE, hanwhaBookingId=rsrvNo)
  → 실패: HanwhaReservationException 발생
  ↓
[5] 예약 응답 로그 저장
  → externalBookingId (한화예약번호) 반환
```

**요청 주요 필드 (ReservationRequest)**

| 필드 | 설명 |
|------|------|
| `allMyTourOrderNumber` | 올마이투어 주문번호 |
| `rateplanIdx` | 요금제 Idx |
| 투숙자 정보 | 이름, 연락처 등 |
| 투숙 정보 | 도착일, 객실수, 연박수 등 |

**한화 예약 요청 주요 필드 (HanwhaReservationRequest)**

| 필드 | 설명 |
|------|------|
| `LOC_CD` | 영업장 코드 |
| `ROOM_TYPE_CD` | 룸타입 코드 |
| `PAKG_NO` | 패키지 번호 |
| `ARRV_DATE` | 도착일 (yyyyMMdd) |
| `OVNT_CNT` | 연박 수 |
| `RSRV_ROOM_CNT` | 예약 객실 수 |
| `INHS_CUST_NM` | 투숙객명 |
| `INHS_CUST_TEL_NO` | 투숙객 연락처 |

---

### 6. 예약 취소 `PUT /v1/realtime/reservation`

올마이투어에서 한화 PMS로 예약 취소를 전송합니다.

**처리 흐름**

```
올마이투어 취소 요청
  → PUT /v1/realtime/reservation (JSON)
  ↓
[1] allMyTourOrderNumber로 hanwha_reservation 조회 (한화예약번호 확인)
  ↓
[2] 예약 취소 요청 로그 저장 (reservation_log)
  ↓
[3] 한화 PMS 취소 API 호출 (CANCEL - HBSREMPRR9902)
  → 한화예약번호(externalBookingId) 기반으로 취소 요청
  ↓
[4] 응답 검증
  MSG_PRCS_RSLT_CD == "0" AND MSG_CD == "SCMI000001"
  → 성공: hanwha_reservation 업데이트 (bookingStatus=CANCELED)
  → 실패: HanwhaReservationException 발생
  ↓
[5] 취소 응답 로그 저장
  → allMyTourOrderNumber, externalBookingId 반환
```

**요청 주요 필드 (ReservationCancelRequest)**

| 필드 | 설명 |
|------|------|
| `allMyTourOrderNumber` | 올마이투어 주문번호 |
| `externalBookingId` | 한화 예약번호 (rsrvNo) |

---

### 7. 예약 조회 `GET /v1/realtime/reservation/{allMyTourOrderNumber}`

올마이투어 주문번호로 한화 PMS에서 예약 정보를 조회합니다.

**처리 흐름**

```
GET /v1/realtime/reservation/{allMyTourOrderNumber}
  ↓
[1] DB에서 hanwha_reservation 조회 → 한화예약번호(rsrvNo) 확인
  ↓
[2] 한화 PMS 조회 API 호출 (RETRIEVE - HBSREMPRR9903)
  → 한화예약번호 기준으로 예약 상세 조회
  ↓
[3] 응답 검증
  MSG_PRCS_RSLT_CD == "0" AND MSG_CD == "SCMI000001"
  → 성공: ReservationSearchResponseInfo 반환
  → 실패: HanwhaReservationException 발생
```

**조회 응답 주요 필드 (DsResult)**

| 필드 | 설명 |
|------|------|
| `RSRV_NO` | 한화 예약번호 |
| `LOC_CD` | 영업장 코드 |
| `ROOM_TYPE_CD` | 룸타입 코드 |
| `ARRV_DATE` | 도착일 |
| `OVNT_CNT` | 연박 수 |
| `RSRV_ROOM_CNT` | 예약 객실 수 |
| `INHS_CUST_NM` | 투숙객명 |
| `INHS_CUST_TEL_NO2~4` | 투숙객 연락처 |

---

### 8. 예약 복구 `POST /v1/realtime/reservation/{allMyTourOrderNumber}`

예약 생성 중 장애 등으로 DB 저장 누락 시 한화 PMS 데이터 기준으로 예약 정보를 복구합니다.

**처리 흐름**

```
POST /v1/realtime/reservation/{allMyTourOrderNumber}
  ↓
[1] DB에서 hanwha_reservation 조회 → 한화예약번호 미존재 확인 (복구 대상 판별)
  ↓
[2] 한화 PMS 조회 API 호출 (RETRIEVE - HBSREMPRR9903)
  → 한화 PMS 실제 예약 정보 조회
  ↓
[3] order_product 조회
  → 올마이투어 주문 정보 확인 (productIdx, roomIdx, rateplanIdx, saleAmount)
  ↓
[4] DB 복구 저장
  한화 PMS 조회 결과 + order_product 기반으로 hanwha_reservation INSERT
  (bookingStatus=COMPLETE, hanwhaBookingId=rsrvNo)
  ↓
[5] externalBookingId 반환
```

---

## 스케줄러

| 스케줄 | Cron | 대상 | 설명 |
|--------|------|------|------|
| 재고 동기화 | `0 0 1 * * *` (매일 01:00) | 전체 매핑 상품 | 케파 API → product_room_stock 업데이트 |
| 요금 동기화 | `0 0 5 * * *` (매일 05:00) | 전체 매핑 상품 | 요금 API → product_rateplan_price 업데이트 |

> 스케줄러는 `prod` 프로파일에서만 동작합니다.

---

## 관련 DB 테이블

| 테이블 | 설명 |
|--------|------|
| `hanwha_product` | 한화 영업장(숙소) 정보 |
| `hanwha_room` | 한화 객실 유형 정보 |
| `hanwha_rate_plan` | 한화 패키지(요금제) 정보 |
| `hanwha_mapping` | 올마이투어 ↔ 한화 상품 매핑 정보 |
| `hanwha_reservation` | 한화 예약 정보 |
| `product_room_stock` | 일별 객실 재고 (케파 동기화 결과) |
| `product_rateplan_price` | 일별 요금제 요금 (요금 동기화 결과) |
| `product_rateplan_price_avail` | 패키지 요금제 가용 여부 |
| `reservation_log` | 예약/취소 요청·응답 로그 |
| `t_product_mapping` | 올마이투어 상품-외부 연동 매핑 |
| `allmy_rateplan_calc_margin_rate` | 요금제별 마진율 설정 |
| `order_product` | 올마이투어 주문 상품 정보 |

---

## 기술 스택

| 항목 | 기술 |
|------|------|
| Framework | Spring Boot |
| HTTP Client | WebClient (Project Reactor / Flux, Mono) |
| 직렬 처리 | `delayElements(500ms)` – 한화 API 병렬 호출 오류 방지 |
| 병렬 처리 | `Flux.parallel() + Schedulers.boundedElastic()` (패키지 상세 조회) |
| ORM | MyBatis |
| 인증 | JWT (Spring Security) |
| DB | MariaDB |
| 스케줄러 | Spring `@Scheduled` |
| 캐시 | Spring Cache (Caffeine) |

---

## 주요 참고사항

- **단일 엔드포인트**: 모든 한화 API는 동일한 URL로 요청하며 `SystemHeader.INTFC_ID`로 서비스를 구분합니다.
- **직렬 처리**: 케파·요금 조회는 한화 API 측의 병렬 오류로 인해 `delayElements(500ms)` 직렬 처리로 전환되었습니다.
- **패키지 상세 조회**: 패키지 목록에서 추출한 패키지번호를 기준으로 Flux 병렬 처리하여 성능을 확보합니다.
- **요금 계산**: 한화에서 받은 `CALC_ROOM_RATE` 기준으로 DB에 저장된 마진율을 적용해 duePrice/netPrice/price를 계산합니다.
- **예약 복구**: 예약 중 장애로 DB 저장 실패 시 `/v1/realtime/reservation/{orderNumber}` POST 호출로 한화 PMS 데이터 기준 복구가 가능합니다.
- **prod 환경 전용**: 스케줄러는 `spring.profiles.active=prod` 일 때만 동작합니다.
