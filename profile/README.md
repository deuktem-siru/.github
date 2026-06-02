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
  <a href="https://github.com/deuktem-siru/buyer-app">Buyer App</a> /
  <a href="https://github.com/deuktem-siru/seller-app">Seller App</a>
</p>

---

## 프로젝트 소개 · 핵심 기능

<p align="center">
  <img src="assets/project-overview.svg" width="100%" alt="득템시루 프로젝트 소개"/>
</p>

<p align="center">
  <img src="assets/core-features.svg" width="100%" alt="득템시루 핵심 기능"/>
</p>

## 시스템 아키텍처

<p align="center">
  <img src="assets/system-architecture.svg" width="100%" alt="득템시루 시스템 아키텍처"/>
</p>

두 앱의 공통 비즈니스 로직을 한 곳에서 일관되게 관리하고, PostgreSQL 트랜잭션으로 주문·정산 데이터의 정합성을 보장하기 위해 이 구조를 선택했습니다.

또한 모니터링을 분리해 작은 팀에서도 운영 상태를 빠르게 파악할 수 있도록 했습니다.

## 앱 화면

### 구매자 앱

<table>
  <tr>
    <th>홈</th>
    <th>찜</th>
    <th>주문 내역</th>
    <th>마이페이지</th>
  </tr>
  <tr>
    <td><img src="assets/BuyerApp/01_home.png" width="200" alt="구매자 앱 홈 화면"/></td>
    <td><img src="assets/BuyerApp/02_wishlist.png" width="200" alt="구매자 앱 찜 화면"/></td>
    <td><img src="assets/BuyerApp/03_orders.png" width="200" alt="구매자 앱 주문 내역 화면"/></td>
    <td><img src="assets/BuyerApp/04_mypage.png" width="200" alt="구매자 앱 마이페이지 화면"/></td>
  </tr>
</table>

### 판매자 앱

<table>
  <tr>
    <th>홈</th>
    <th>주문 관리</th>
    <th>매출 분석</th>
    <th>상품 관리</th>
  </tr>
  <tr>
    <td><img src="assets/SellerApp/01_home.png" width="200" alt="판매자 앱 홈 화면"/></td>
    <td><img src="assets/SellerApp/02_orders.png" width="200" alt="판매자 앱 주문 관리 화면"/></td>
    <td><img src="assets/SellerApp/03_sales.png" width="200" alt="판매자 앱 매출 분석 화면"/></td>
    <td><img src="assets/SellerApp/04_products.png" width="200" alt="판매자 앱 상품 관리 화면"/></td>
  </tr>
</table>

## 포스터

<p align="center">
  <img src="https://img.shields.io/badge/2026학년도%201학기%20지역사회참여교과%20우수사례-FFB300?style=for-the-badge&logoColor=white" alt="2026학년도 1학기 지역사회참여교과 우수사례"/>
</p>

<p align="center">
  <img src="assets/poster.webp" width="480" alt="득템시루 프로젝트 포스터"/>
</p>

## 팀 구성

<table>
  <tr><td align="center"><a href="https://github.com/Asterisk0707"><img src="https://github.com/Asterisk0707.png" width="60px" alt="한동혁"/></a></td><td><b>한동혁</b><br/><sub>팀장 · 프론트엔드</sub></td><td>구매자 앱 (지도 SDK, 장바구니·찜, 경로 안내)</td></tr>
  <tr><td align="center"><a href="https://github.com/ken-jeong"><img src="https://github.com/ken-jeong.png" width="60px" alt="정상겸"/></a></td><td><b>정상겸</b><br/><sub>기획 · 풀스택</sub></td><td>구매자·판매자 앱 및 백엔드 연동, 테스트·모니터링, 문서 작성</td></tr>
  <tr><td align="center"><a href="https://github.com/Movinggun-bit"><img src="https://github.com/Movinggun-bit.png" width="60px" alt="이동건"/></a></td><td><b>이동건</b><br/><sub>풀스택 · 인프라</sub></td><td>백엔드 ERD·API 설계, Kakao 로그인, AWS 배포</td></tr>
  <tr><td align="center"><a href="https://github.com/yeonwuyoo"><img src="https://github.com/yeonwuyoo.png" width="60px" alt="유연우"/></a></td><td><b>유연우</b><br/><sub>프론트엔드 · PPT</sub></td><td>판매자 앱 (픽업 코드, 상품관리·고객 알림), 발표 자료</td></tr>
  <tr><td align="center"><a href="https://github.com/Ra-seungmin"><img src="https://github.com/Ra-seungmin.png" width="60px" alt="라승민"/></a></td><td><b>라승민</b><br/><sub>프론트엔드</sub></td><td>구매자 앱 (UI)</td></tr>
</table>
