<h1 align="center">득템시루</h1>

<p align="center">
  <b>오늘 팔리지 않으면 버려지는 상품을 가까운 소비자에게 빠르게 연결합니다</b><br/>
  시흥시 지역화폐 시루 기반 마감 할인 픽업 커머스 프로젝트
</p>

<p align="center">
  <img src="https://img.shields.io/badge/Kotlin-7F52FF?style=for-the-badge&logo=kotlin&logoColor=white" alt="Kotlin"/>
  <img src="https://img.shields.io/badge/Android-3DDC84?style=for-the-badge&logo=android&logoColor=white" alt="Android"/>
  <img src="https://img.shields.io/badge/Spring Boot-6DB33F?style=for-the-badge&logo=springboot&logoColor=white" alt="Spring Boot"/>
  <img src="https://img.shields.io/badge/PostgreSQL-4169E1?style=for-the-badge&logo=postgresql&logoColor=white" alt="PostgreSQL"/>
</p>

<p align="center">
  <a href="https://github.com/deuktem-siru/backend">Backend</a> /
  <a href="https://github.com/deuktem-siru/android-app">Android</a>
</p>

---

## 핵심 기능

<p align="center">
  <img src="../assets/key-features.svg" width="100%" alt="득템시루 핵심 기능"/>
</p>

## 시스템 아키텍처

<p align="center">
  <img src="../assets/architecture.svg" width="100%" alt="득템시루 시스템 아키텍처"/>
</p>

두 앱은 하나의 백엔드를 사용하며 PostgreSQL 트랜잭션으로 주문·정산 데이터의 정합성을 관리합니다. Prometheus·Grafana로 운영 상태를 확인합니다.

Android 코드는 [android-app](https://github.com/deuktem-siru/android-app) 저장소의 구매자 `buyer`·판매자 `seller`·공통 `common` 모듈로 통합했습니다. 인증·카카오 로그인·암호화 저장소·푸시 처리를 공유하며 앱은 각각 설치합니다.

백엔드는 헥사고날·클린 아키텍처의 유스케이스·출력 포트·어댑터로, Android는 MVVM·클린 아키텍처의 View·ViewModel·도메인·데이터 구현으로 분리합니다. JPA 매핑과 일부 기존 화면 렌더링은 유지합니다. 정확한 의존성 방향과 검사 범위는 [아키텍처 설계](../plan.md#아키텍처)를 참고하세요.

## 앱 화면

2026-10-28 현재 소스에서 구매자·판매자 Debug APK를 캐시 재사용 없이 다시 빌드·설치한 뒤 Android 에뮬레이터에서 실제 앱 화면 8장을 새로 촬영했습니다. 구매자 앱은 캡처 전용 로컬 개발 DB의 샘플 데이터와 시루 내부 잔액 시뮬레이션, 판매자 앱은 내장 `mock` 데모 모드를 사용했습니다.

탭 목록 대신 **탐색 → 상품 선택 → 결제 → 픽업**, **상품 관리 → 주문 수락 → 픽업 검증 → 매출 확인**의 핵심 흐름을 선정했습니다.

### 구매자 앱

| 주변 할인 매장 탐색 | 매장·상품·픽업 시간 | 시루 결제 | 주문 수락 후 픽업 |
| --- | --- | --- | --- |
| <img src="../assets/BuyerApp/01_home.png" width="220" alt="주변 매장의 할인율과 남은 재고 탐색"/> | <img src="../assets/BuyerApp/02_store.png" width="220" alt="매장 정보와 할인 상품 및 픽업 시간 확인"/> | <img src="../assets/BuyerApp/03_payment.png" width="220" alt="시루 보유 잔액과 결제 후 잔액 확인"/> | <img src="../assets/BuyerApp/04_pickup.png" width="220" alt="판매자 수락 후 픽업 코드와 주문 정보 확인"/> |

### 판매자 앱

| 상품·재고 관리 | 신규 주문 수락 | 픽업 코드 검증 | 매출 분석 |
| --- | --- | --- | --- |
| <img src="../assets/SellerApp/04_products.png" width="220" alt="상품별 잔여 수량과 판매 상태 관리"/> | <img src="../assets/SellerApp/02_orders.png" width="220" alt="신규 주문의 메뉴와 픽업 시간을 확인하고 수락"/> | <img src="../assets/SellerApp/01_pickup_verify.png" width="220" alt="픽업 코드 검증 후 고객과 주문 정보 확인"/> | <img src="../assets/SellerApp/03_sales.png" width="220" alt="기간별 매출과 판매 실적 분석"/> |

## 포스터

<p align="center">
  <img src="https://img.shields.io/badge/2026학년도%201학기%20지역사회참여교과%20우수사례-FFB300?style=for-the-badge&logoColor=white" alt="2026학년도 1학기 지역사회참여교과 우수사례"/>
</p>

<p align="center">
  <img src="../assets/poster.webp" width="480" alt="득템시루 프로젝트 포스터"/>
</p>

## 팀 구성

<table>
  <tr><td align="center"><a href="https://github.com/Asterisk0707"><img src="https://github.com/Asterisk0707.png" width="60px" alt="한동혁"/></a></td><td><b>한동혁</b><br/><sub>팀장 · 프론트엔드</sub></td><td>구매자 앱 (지도 SDK, 장바구니·찜, 경로 안내)</td></tr>
  <tr><td align="center"><a href="https://github.com/ken-jeong"><img src="https://github.com/ken-jeong.png" width="60px" alt="정상겸"/></a></td><td><b>정상겸</b><br/><sub>기획 · 풀스택</sub></td><td>구매자·판매자 앱 및 백엔드 연동, 테스트·모니터링, 문서 작성</td></tr>
  <tr><td align="center"><a href="https://github.com/Movinggun-bit"><img src="https://github.com/Movinggun-bit.png" width="60px" alt="이동건"/></a></td><td><b>이동건</b><br/><sub>풀스택 · 인프라</sub></td><td>백엔드 ERD·API 설계, Kakao 로그인, AWS 배포</td></tr>
  <tr><td align="center"><a href="https://github.com/yeonwuyoo"><img src="https://github.com/yeonwuyoo.png" width="60px" alt="유연우"/></a></td><td><b>유연우</b><br/><sub>프론트엔드 · PPT</sub></td><td>판매자 앱 (픽업 코드, 상품관리·고객 알림), 발표 자료</td></tr>
  <tr><td align="center"><a href="https://github.com/Ra-seungmin"><img src="https://github.com/Ra-seungmin.png" width="60px" alt="라승민"/></a></td><td><b>라승민</b><br/><sub>프론트엔드</sub></td><td>구매자 앱 (UI)</td></tr>
</table>
