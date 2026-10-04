# PartnerBase

기업 정보와 협업 수요를 기반으로 거래처를 탐색하는 B2B 플랫폼의 초기 프로젝트입니다.

## 설치와 실행

```bash
npm install
npm run dev
```

프로덕션 빌드는 `npm run build`로 확인합니다.

## Firebase 설정

배포 주소: https://partnerbase.web.app

`.env.example`을 복사해 `.env`를 만들고 Firebase 웹 앱 설정 값을 입력합니다. `.env`는 Git에서 제외됩니다.

```env
VITE_FIREBASE_API_KEY=
VITE_FIREBASE_AUTH_DOMAIN=
VITE_FIREBASE_PROJECT_ID=
VITE_FIREBASE_STORAGE_BUCKET=
VITE_FIREBASE_MESSAGING_SENDER_ID=
VITE_FIREBASE_APP_ID=
```

외부 기업 데이터는 공공데이터포털 등에서 발급받은 서비스 키를 Cloud Functions의 서버 환경변수로만 설정합니다. 브라우저에 키를 노출하는 `VITE_` 환경변수에는 넣지 않습니다.

설정값이 없는 경우 Firebase 인스턴스를 만들지 않으므로 앱 화면은 안전하게 열립니다. 인증이나 데이터 저장을 실행하면 설정 안내 오류를 보여줍니다.

## 구조

- `src/components`: 공통 UI와 기업 카드
- `src/pages`: 공개/보호 화면
- `src/layouts`: Main, Dashboard 레이아웃
- `src/contexts`: 인증 상태
- `src/services`: Firebase 및 도메인별 호출
- `src/types`: User, Company, RFQ, Quote, Connection 타입
- `src/routes`: React Router와 ProtectedRoute

## 현재 구현

- 이메일/비밀번호 Firebase Authentication 구조와 AuthContext
- 환경변수 기반 Firebase modular SDK 초기화
- 기업 목록·상세·검색 UI, 회사 프로필 태그 입력 폼
- 대시보드 및 RFQ/견적/거래처 기본 화면
- Firestore 저장 함수의 `serverTimestamp()` 사용
- 견적에 구매사·공급사 회사 ID를 명시해 향후 비공개 Security Rules 적용 가능
- 기업 피드(`/feed`): 공급·구매 수요·생산 여력·기업 소식 게시, 팔로우, 업체 저장, 명함 보내기
- 회사 프로필의 활동 탭 및 게시글에서 시작하는 견적·공급·협업·미팅 문의
- 기업 문의함(`/inquiries`): 구매사·공급사 소속 사용자만 문의 열람

피드 글은 PartnerBase 등록 기업에 소속된 로그인 사용자가 회사 이름으로 작성합니다.
외부 공공데이터·OpenDART 기업은 글을 올린 회원사로 표시하지 않습니다.
문의 내용과 수량은 `postInquiries`에 저장하고 공개 피드에 노출하지 않습니다.
실제 견적 금액 제출·수락 절차는 기존 RFQ/견적 기능에 남아 있으며, 피드 문의함은 초기 연락 단계입니다.

## OpenDART 기본정보 추가 수집

`node scripts/collectDartProfiles.mjs`를 실행하면 종목코드가 있는 기업 중 JSON에 아직 없는 기업을 수집합니다.
배포된 `dartCompanyProfileV2` 함수가 Firestore에 저장된 정보를 먼저 반환하고, 없을 때만 OpenDART를 호출해 저장합니다.
수집 결과는 `public/data/dart-company-profiles.json`에 50건마다 원자적으로 저장됩니다.
재실행 시 저장된 기업을 건너뛰며, 연속 실패 시 중단해 무분별한 재호출을 방지합니다.
갱신한 JSON을 사이트에 반영하려면 빌드 후 Hosting에 배포합니다.

## 공공 사업자 CSV 등록

공공 사업자 CSV의 DB 등록은 `node scripts/importPublicBusinesses.mjs`로 사전 검증한 뒤
`node scripts/importPublicBusinesses.mjs --write`로 실행합니다. Firebase CLI 로그인 계정을 사용합니다.
`src/data`의 통신판매업·튜닝정비·전문연구·방송·근로자공급 자료를 대상으로 하며,
`publicBusinessProfiles`에 출처와 기준일, 원본 행을 저장합니다. 통계와 중복 파일은 가져오지 않습니다.
사업자번호가 없는 경우 출처·이름·주소·지역 기준으로 식별하므로 서로 다른 출처의 동일 사업자는 별도로 남을 수 있습니다.
기존 `companies` 가입 기업 및 OpenDART 기업과 별도 섹션으로 기업 검색 UI에 표시합니다.
이름·업종·주소 검색, 업종/지역 필터, 24건씩 더 보기와 출처·기준일이 있는 상세 화면을 제공합니다.

DB 갱신 후 `node scripts/exportPublicBusinessDirectory.mjs`로 공개 검색 목록을 추출합니다.
`public/data/public-businesses.json`에는 표시용 필드만 포함하며, 원본 행·대표자·사업자번호는 내보내지 않습니다.
목록은 방문자마다 Firestore 전체를 조회하지 않도록 Hosting에서 제공하며, 자동 실시간 동기화는 아닙니다.
추출 후 `npm run build` 및 `npx firebase deploy --only hosting --project velder-381f3`로 반영합니다. Hosting 대상은 `firebase.json`의 `partnerbase` 사이트입니다.

## 공정거래위원회 통신판매업 API 수집

공식 명세: https://www.data.go.kr/data/15126311/openapi.do

`MllBs_2Service/getMllBsInfo_2`를 Node에서만 호출합니다. Git에서 제외되는
`.env.mll.local`에 `DATA_GO_KR_SERVICE_KEY`를 설정합니다. 키는 브라우저·배포 JSON에 포함하지 않습니다.

```sh
# 첫 페이지 검증 (DB 변경 없음)
node --env-file=.env.mll.local scripts/syncMllBusinesses.mjs
# 정상영업 자료 100건씩 최대 10페이지 저장, 재실행하면 다음 페이지부터 재개
node --env-file=.env.mll.local scripts/syncMllBusinesses.mjs --write --max-pages=10
node --test scripts/lib/mllApi.test.mjs
```

Firebase CLI 로그인 계정으로 `publicBusinessProfiles`에 저장합니다.
사업자번호가 있으면 기존 CSV와 같은 문서 ID를 사용하고, 없으면 인허가관리번호로 식별합니다.
기존 업종·연락처 등은 유지하며 출처를 병합합니다. 여러 신고가 같은 사업자번호를 가지면 한 문서에 출처 기록을 합칩니다.
관리부서 전화번호를 업체 전화번호로 표시하지 않으며, 대표자명·이메일은 수집 파일에 보관하지 않습니다.

`.local-data/mll-normal-v1`에 비공개 페이지 파일과 DB 저장 완료 체크포인트를 유지합니다.
실패한 페이지는 저장 파일에서 재시도하며, 동시 DB 변경이 있으면 덮어쓰지 않고 중단합니다.
API 페이지 순서가 갱신 중 변동할 수 있어 이 수집만으로 전국 자료의 누락 없는 동기화를 보장하지 않습니다.
전체 수백만 건을 단일 공개 JSON으로 내보내지 마세요. 대량 수집 전 서버 검색·증분 갱신 및 비용 한도를 설계해야 합니다.
현재는 제한된 초기 수집이며, 정기 자동 수집은 설정하지 않았습니다.
DB 저장 후 위의 공개 목록 추출 및 Hosting 배포 절차로 검색 화면에 반영합니다.

사업자 상태조회는 `verifyBusinessStatus` callable 함수로 처리합니다.
`functions/.env`에도 `DATA_GO_KR_SERVICE_KEY`를 설정한 뒤 해당 함수를 배포하세요.
로그인이 필요하며 사용자별 분당 5회로 제한합니다. 기존 `VITE_ODCLOUD_BUSINESS_API_KEY`는 제거했습니다.
과거 프런트엔드에 포함된 키는 새 배포만으로 노출 이력이 사라지지 않으므로 포털에서 재발급 후 서버 설정을 교체하세요.

## 다음 개발 우선순위

1. 기업 가입 및 인증
2. 기업 검색
3. 협업 요청
4. RFQ 생성
5. 비공개 견적
6. 견적 비교
7. 거래처 관리
8. 기업 추천 알고리즘
