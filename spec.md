# SPEC — 득템시루

## 개요

마감 임박 식품을 지역 픽업으로 연결하는 구매자·판매자 Android 서비스. 시흥시 지역화폐 시루 결제를 전제로 한다.

## 범위

- `CONSUMER`: 탐색·장바구니·주문/취소·찜·리뷰·알림. `SELLER`: 매장·메뉴·상품·주문·픽업 검증·매출·알림·정산 신청(`sellers/**` 전용).
- 시루는 외부 API 없는 시뮬레이션(연동 플래그와 초기 잔액 50,000, DB 정수 잔액). 사업자 API 미발급으로 실연동 계획은 없다.
- 제외: 배달·택배, 앱 내 카드/현금 실결제, iOS·웹, 관리자 콘솔.

## 요구사항

- 카카오 단일 로그인·최초 자동 가입. debug 로그인은 dev 전용. Bearer JWT 만료 시 `/auth/refresh`로 갱신하며 응답은 `ApiResponse<T>`.
- 상품: `AVAILABLE ↔ PAUSED`, 잔여 0이면 `SOLD_OUT`, 마감 경과 `EXPIRED`, 삭제 API로 `DELETED`.
- 주문: `PENDING → CONFIRMED → PICKED_UP`; `PENDING/CONFIRMED → CANCELLED`.
- 매장 분류: `BAKERY/RESTAURANT/CAFE/GROCERY/OTHER`. 결제: `SIRU/CARD/CASH` × `PENDING/COMPLETED/FAILED/REFUNDED`.
- 시루는 연동·잔액 충족 시에만 차감하고 취소 시 환급한다. 동시 주문으로 재고를 초과 판매하지 않는다.
- 리뷰는 본인의 `PICKED_UP` 주문당 1개, 별점 1~5. 픽업은 판매자의 코드/QR 검증으로 확정한다. 정산 수수료율은 3%.
- 이미지는 확장자·매직넘버·경로 탈출을 검증한다. JWT는 암호화 저장하고, 릴리스는 HTTPS `BACKEND_BASE_URL`이 아니면 기동을 차단한다.
- `local.properties`·`google-services.json`·`release.keystore`는 커밋하지 않는다.
- 운영 준비 요구: 미승인 매장의 상품 등록·판매 차단, 사업자/매장 승인·반려, 정산 지급·반려와 앱 상태 표시, 인스턴스 교체에도 이미지 유지.

## 완료 기준

- 주문·취소 환급·잔액 부족/미연동 롤백·동시 재고·리뷰 제한·픽업 검증과 prod 안전장치 검사를 통과한다.
- 운영 준비 후 승인 전후 상품 등록과 정산 `신청 → 지급/반려`를 재현하고, 교체된 인스턴스에서도 이미지를 조회한다.
- main 푸시로 EC2의 새 JAR·헬스체크가 성공하며, 두 앱의 실기기 로그인→주문→픽업→푸시가 통과한다.
