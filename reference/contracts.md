# 득템시루 API 계약

[요구사항](../spec.md) · [구현 계획](../plan.md) · [데이터 모델](data-model.md)

## 기준과 범위

구매자·판매자 앱이 서버와 연동할 때 필요한 공통 규약과 호출 흐름을 기록한다. 전체 엔드포인트 목록과 DTO 필드·타입·필수값은 대상 서버의 Swagger UI(`/swagger-ui/index.html`)와 `/v3/api-docs`에서 확인한다. 로컬은 [Swagger UI](http://localhost:8080/swagger-ui/index.html)·[OpenAPI JSON](http://localhost:8080/v3/api-docs), 실행 방법은 [Backend README](https://github.com/DeuktemSiru/Backend/blob/main/README.md)를 따른다.

아래는 현재 구현의 동작과 제약이다. 미구현 요구사항과 진행 상태는 `spec.md`·`plan.md`·`tasks.md`에서 관리한다. Swagger의 요약만으로 상태 전이나 서비스 검증 전체를 판단하지 않는다.

## 공통 규약

| 항목 | 규약 |
| --- | --- |
| API 루트 | 로컬 `http://localhost:8080/api/v1`, 에뮬레이터 `http://10.0.2.2:8080/api/v1` |
| 앱 설정 | `local.properties`의 `BACKEND_BASE_URL`은 호스트 루트. 앱 요청 경로에 `api/v1`이 포함된다. |
| 본문 | 기본 `application/json`, 이미지 첨부는 `multipart/form-data` |
| 인증 | `Authorization: Bearer {accessToken}` |
| 단위 | 금액은 정수 원, `distanceM`·`radius`는 m, `radiusKm`는 km |
| 날짜·시간 | 날짜 `yyyy-MM-dd`, 시간은 `LocalTime` 파싱 형식(예: `18:30`). 서버 기본 시간대를 사용하며 KST로 고정하지 않는다. |
| 페이지 | 기본 `page=0`, `size=20`. 주문·리뷰 size는 1~100, 매장·상품 목록은 최솟값 1만 보정한다. |

이 문서의 `/auth`, `/orders`, `/sellers` 등 업무 경로에는 `/api/v1`을 붙인다. Swagger·Actuator·업로드 경로는 호스트 루트 기준이다.

### 인증과 권한

- 카카오 로그인과 토큰 갱신은 공개다. 매장·상품 조회도 로그인이 필요하다.
- `/sellers/**`는 `SELLER`, `/cart/**`·`/orders/**`·`/wishlist/**`는 `CONSUMER` 전용이다. 나머지 업무 API는 기본적으로 로그인 필요다.
- 회원 ID는 JWT에서 얻고 URL의 매장·상품·주문 ID는 대상 리소스를 가리킨다. 서비스에서 본인 소유 여부를 추가 검사한다.
- Swagger·`/v3/api-docs/**`·`/actuator/health`·`/uploads/**`는 공개다. 디버그 로그인과 `/actuator/prometheus`는 개발 API 활성화 시에만 공개한다.
- `/admin/**`은 JWT 대신 `X-Admin-Token` 헤더로 인증한다. `app.admin.token`(`APP_ADMIN_TOKEN`)이 32자 미만이면 운영자 API 전체가 401이다.

### 응답과 오류

일반 응답은 `ApiResponse<T>`다.

```json
{ "code": 200, "message": "성공", "data": {} }
```

통상 성공은 200, 생성은 201이며 개별 성공 코드는 Swagger를 확인한다. 공통 오류 응답의 `data`는 null이다. Kotlin `Unit` 반환의 직렬화 형태는 런타임 확인 대상이므로 빈 객체나 null로 단정하지 않는다.

| 상황 | 처리 |
| --- | --- |
| 입력·조건 검증 실패 (`require`/`IllegalArgumentException`) | 400. 소유권 불일치·재고 부족·리뷰 중복 포함 |
| `UnauthorizedException` | 401 |
| `NoSuchElementException` | 404 |
| 상태 충돌 (`check`/`IllegalStateException`) | 409 |
| 그 외 | 500, `서버 오류가 발생했습니다.` |
| 보안 필터의 미인증·역할 거부 | 공통 JSON 형식을 보장하지 않음 |

JSON 파싱 실패·필수 파라미터 누락은 별도 매핑이 없어 400을 보장하지 않는다. HTTP 상태와 본문 `code`가 항상 같다고 가정하지 않는다. 찜 토글은 추가 시 HTTP 201이지만 본문 `code`는 200이다.

## 인증과 회원

1. `POST /auth/kakao/login`으로 로그인한다. 기존 회원은 200, 신규 자동 가입은 201이다. 기존 계정의 역할은 요청의 `role`로 바꾸지 않으며 닉네임·프로필 이미지만 갱신한다. 카카오 토큰 검증 실패는 400이다.
2. 액세스 토큰 만료 시 `POST /auth/refresh`로 새 액세스 토큰을 받는다. 리프레시 토큰 자체는 회전하지 않는다. 서명·타입·만료·DB 폐기 여부 검증 실패는 401이다. 기본 만료는 액세스 30분, 리프레시 14일이다.
3. `POST /auth/logout`은 회원의 리프레시 토큰 전체를 폐기하고 FCM 토큰을 비활성화한다. 기존 액세스 토큰은 즉시 폐기하지 않는다.

디버그 로그인은 `app.security.dev-endpoints-enabled=true`일 때만 사용한다. `role`은 필수, `email`은 판매자 샘플 선택용이다. 비활성 상태에서 404가 반드시 반환되는 것은 아니며 보안 필터가 먼저 요청을 막을 수도 있다.

시루 연동은 외부 검증 없이 전달값의 공백 여부만 확인한다. 연동 시 내부 잔액이 0이면 50,000을 부여하고, 해제 시 연동 상태와 잔액을 초기화한다. 외부 시루 계정·잔액과 동기화되지 않는다.

프로필 수정은 빈 닉네임을 무시하고 최대 30자를 받는다. 탈퇴는 비활성화·탈퇴 시각 기록이다. 알림 설정에서 null은 기존 값 유지이며 해당 역할과 공통 `event`만 수정한다. 다른 역할의 설정은 응답에서 null이다.

회원 통계의 등급은 주문 수 0~4 `SEEDLING`, 5~14 `SPROUT`, 15~29 `TREE`, 30 이상 `FOREST`다. `points`는 절약 금액÷10의 정수값, `couponCount`는 0 고정이다. DB의 회원 활성 여부와 사업자 인증 여부가 Boolean이어도 해당 응답 필드는 정수 1/0으로 표현된다.

## 탐색과 장바구니

- 승인(`isVerified`)되지 않은 매장은 목록·지도·상품 목록에서 제외하고 매장 상세는 404다. 미승인 매장의 상품은 상세 조회·장바구니·주문에서 400이다.
- 매장 경로는 `/stores`다. 목록 `sort=rating`은 평점 내림차순 후 거리순, `products`/`available`은 가용 재고 합계 내림차순이다. 상품 `sort=discount`는 **할인가 오름차순**, `quantity`는 잔여 수량 내림차순이다. 기본은 거리순이며 알 수 없는 값도 기본 정렬을 따른다.
- 매장·상품 목록은 `hasNext`를 반환한다. 리뷰 목록에는 전체 평균·리뷰 수와 해당 페이지가 함께 온다.
- 매장·찜 목록의 `availableProductCount`는 오늘 판매 가능한 상품의 잔여 수량 합계다. 지도 마커의 상품 개수 집계와 다르다.
- `GET /stores/map`은 위경도를 필수로 받지만 서비스에 category만 전달하므로 현재 위치·반경 필터가 적용되지 않는다.
- `POST /wishlist/{storeId}`는 토글이다. 해제 의도를 명확히 전달하려면 `DELETE /wishlist/{storeId}`를 사용한다.
- 장바구니에는 같은 매장의 오늘 판매 가능한 `AVAILABLE` 상품만 담는다. 동일 상품은 수량을 합산하고 양수 수량·재고를 검사한다. 담는 시점에는 재고를 차감하지 않는다.

## 주문·취소·픽업

1. 구매자가 `POST /orders`를 호출한다. 비어 있지 않은 `items`에 양수 수량을 보내며 같은 상품은 서버가 합산한다. 같은 매장의 오늘 구매 가능한 상품만 허용하고 주문 시 재고를 차감한다.
2. 선택한 `pickupTime`은 모든 주문 상품의 픽업 가능 시간 안이어야 한다. `paymentMethod` 생략/null은 `CASH`다. 모든 결제수단은 내부적으로 `COMPLETED`로 기록하고 외부 승인을 호출하지 않는다. `SIRU`는 연동 여부와 내부 잔액을 검사해 차감한다.
3. 생성된 주문은 `PENDING`이다. 장바구니는 자동으로 비워지지 않는다. 픽업 코드는 보통 영문 대문자·숫자 6자리지만 충돌 시 8자리가 될 수 있다.
4. 판매자가 주문을 `CONFIRMED`로 변경한 뒤 `GET /sellers/pickup/verify`로 코드를 조회한다. 검증은 해당 매장의 확정 주문을 조회할 뿐 픽업을 완료하지 않는다.
5. `PATCH /sellers/orders/{orderId}/confirm`은 주문 ID와 코드를 함께 검사해 `PICKED_UP`으로 바꾼다. 일반 주문 상태 변경 API로도 픽업 완료 전이가 가능하다.

허용 전이는 `PENDING → CONFIRMED/CANCELLED`, `CONFIRMED → PICKED_UP/CANCELLED`다. 판매자 상태 변경은 같은 상태 재요청을 허용하지만 완료·취소 후 다른 상태로 바꿀 수 없다. 구매자 취소도 `PENDING`·`CONFIRMED`에서만 가능하며 재고·시루 잔액을 복원하고 환불을 기록한다.

리뷰는 본인의 `PICKED_UP` 주문과 해당 매장에만 작성할 수 있고 별점 1~5, 회원·주문당 1건이다. 삭제는 논리 삭제 후 평점을 재집계한다.

## 판매 상품

- 사업자 등록은 번호의 공백 여부만 확인하고 미인증으로 저장한다. 외부 인증 호출은 없고 승인은 운영자 API로 처리한다.
- 상품 등록은 매장이 승인된 뒤에만 가능하다. 미승인 매장이 등록을 시도하면 400이다.
- 메뉴는 이름과 1원 이상의 정상가가 필요하고 삭제 시 비활성화한다. 상품의 메뉴 참조는 선택이며 지정하면 설명·기본 이미지·알레르기 정보를 가져온다.
- 상품 등록은 총수량 > 0, `0 < discountPrice < originalPrice`, 픽업 종료 > 시작을 요구한다.
- 상품 상세 수정은 정상가·할인가·잔여 수량을 지원한다. 잔여 수량은 0~총수량이며 0이면 `SOLD_OUT`, 품절 상품에 양수를 넣으면 `AVAILABLE`로 돌아간다.
- 상태 API로 직접 지정할 수 있는 값은 `AVAILABLE`·`PAUSED`·`EXPIRED`다. 재고가 없으면 `AVAILABLE`/`PAUSED` 전환을 거부한다. 삭제는 `DELETED` 전이이며 진행 중 주문(`PENDING`/`CONFIRMED`)이 있으면 거부한다.

## 이미지 업로드

매장·메뉴·상품 등록은 JSON 또는 multipart를 받는다. 파일 필드는 매장 `thumbnail`·`images`, 메뉴 `image`, 상품 `images`다. 일반 필드는 폼 필드로 바인딩하므로 DTO 전체를 단일 JSON 파트로 보내지 않는다.

파일당 5MB, 요청 전체 6MB이며 MIME·확장자·파일 시그니처를 검사한다. 현재 반환 이미지는 `/uploads/menu-images/...`에서 제공한다. 현재 저장 위치와 배포 절차는 [배포 문서](deploy.md), S3 전환·이관 계획은 [구현 계획](../plan.md)을 따른다.

## 매출·정산·알림

- 매출은 픽업 완료 주문을 **주문 생성일 기준**으로 집계한다. `period`는 `DAY`·`WEEK`·`MONTH`·`YEAR`, 알 수 없는 값은 `DAY`이며 `carbonSavedKg`는 0.0 고정이다.
- 정산은 월 1~12를 받으며 픽업 완료 매출에서 수수료 3%(반올림)를 뺀다. 현재 월에는 `settlementId=0`인 계산 항목이 포함될 수 있으므로 저장된 정산 ID로 간주하지 않는다.
- 출금 신청은 `PENDING` 정산을 생성하거나 기존 대기 레코드를 반환하며 실제 송금은 하지 않는다.
- 정산 상태는 `PENDING`·`COMPLETED`·`REJECTED`다. 운영자 API로 `PENDING`에서만 전이하며 `COMPLETED`는 `settledAt`을 기록한다. 이미 처리된 정산에 다시 전이하면 409다. 반려된 달은 다시 출금 신청할 수 있다.
- 판매자 알림 `REGULAR`의 수신자는 찜 회원과 주문 이력이 있는 활성 회원을 합친 중복 제거 목록이다. `NEARBY`는 회원 위치가 없어 409다. `radiusKm`를 주면 1 이상이어야 한다.
- `recipientCount`는 선정된 수신자 수이며 기기 전달 성공 수가 아니다. 구매자 알림 목록에는 페이지 파라미터가 없다.

## 운영자 API

관리자 콘솔이 범위 밖이라 운영자가 아래 엔드포인트를 직접 호출한다. 모든 요청에 `X-Admin-Token: {APP_ADMIN_TOKEN}` 헤더가 필요하다.

| 엔드포인트 | 본문 | 동작 |
| --- | --- | --- |
| `POST /admin/business-infos/{businessInfoId}/verification` | `{ "approved": true, "siruApproved": true }` | 사업자 승인/반려. `approved=true`면 `verifiedAt` 기록, `false`면 `verifiedAt`·`isSiruVerified` 초기화. `siruApproved`가 null이면 시루 인증 상태를 유지한다. |
| `POST /admin/stores/{storeId}/verification` | `{ "approved": true }` | 매장 승인/반려. 승인해야 상품 등록·판매가 열린다. |
| `POST /admin/settlements/{settlementId}/status` | `{ "status": "COMPLETED" }` | 정산 지급 완료/반려(`REJECTED`). 실제 송금은 수기로 처리한다. |

토큰 불일치·미설정은 401, 대상이 없으면 404, 이미 처리된 정산은 409다.

## 변경할 때

경로·DTO 변경은 컨트롤러와 DTO에 반영하고 `/v3/api-docs`에서 확인한다. 인증·상태 전이·부수 효과·해석이 달라지면 이 문서를 갱신하고 두 앱의 호출·응답 모델을 맞춘다. DB 변경은 [데이터 모델](data-model.md)의 기준을 따른다. 서버와 앱은 독립 저장소이므로 관련 변경의 배포 순서도 함께 확인한다.
