# Tasks Log

---

## 2026-10-28

### [MCI] isBiz 파라미터 제거 및 구조 수정 (PL 피드백)
- **상태**: 해결
- **문제**: fcccb1100tS.execute()에 isBiz 파라미터를 추가했으나, 컨트롤러(@McuCallID)에서 이미 라우팅하므로 서비스에서 중복 체크 불필요
- **원인**: 사업자/직장인 분기를 서비스 레이어에서도 했으나, 이미 컨트롤러에서 분리됨
- **해결**:
  1. fcccb1100tS.execute()의 isBiz 파라미터 전부 제거
  2. fctcb1100tS.reqLoanLimitTc에서 jonCls="5" 사업자 분기 추가 (isWorker 앞에)
  3. reqLoanLimitBizTc 삭제, reqLoanLimitTc 하나로 통합
  4. 사업자 분기 안에서 4필드 세팅 + callBusinessMciEai 호출
- **배운 점**: 컨트롤러에서 이미 라우팅하는 구조면 서비스에 중복 분기 플래그 넣지 말 것. 기존 requestMap의 값(jonCls)을 활용

### [MCI] 상품코드 PD301 직접 매핑 (ps110 ROWNUM 우회)
- **상태**: 해결
- **문제**: ps110 쿼리 ROWNUM=1으로 PD001이 먼저 잡힘 (PD301이 나가야 함)
- **원인**: 쿼리 정렬 순서상 PD001이 먼저 리턴
- **해결**: setMciBody에서 jonCls="5" 분기 추가, TIUtil.getEffcCore("aflcnC", aflcnC)로 yml에서 PD301 직접 매핑. common_ti_config.yml에 aflcnC: "fco0001": PD301 추가
- **배운 점**: 값이 고정인 경우 쿼리 결과보다 설정값이 안전. DB 의존 대신 yml 외부화

### [MCI] 사업자 MCI 송수신 테스트 결과
- **상태**: 부분해결
- **결과**:
  - 로컬: MCI 송신 정상, 역거래 수신 정상, setMciBody PD301 확인
  - 배포환경(dev): MCI 가심사요청 송신 정상 확인
  - 역거래 수신은 정상이나 제휴사 결과값 미유입으로 유효성 필수값 오류코드 수신
- **대기**: 황인준 과장 스크래핑 진입점 연결 후 실데이터 재테스트 필요

### [BRIDGE] 당행 대출신청 bridge_common 수정
- **상태**: 대기
- **내용**: PL이 bridge_common.js 수정 후 배포 진행 중 (APP/SB 분기 제거 → opa.execute 통일)
- **테스트 결과**: MT 1초 뒤 원복, PT opa.execute 응답 없이 멈춤 — 로컬/PC 환경 한계
- **대기**: predev 배포 후 실기기 모바일 테스트 필요

### [MCI] 사업자 4필드 빈값 이슈
- **상태**: 조사중
- **문제**: MCI 전문에 사업자번호(io_dh_business_id)만 값 있고, 공동사업자여부/납세자형태/체납여부 3개 빈값
- **원인 추정**: 건보인증/홈택스 스크래핑을 테스트에서 패스 → TB_EF_FC_CH_HTAX_PSN_BZMN 테이블 미적재
- **추가 발견**: scrpgSucYn 강제 "Y" 원복했는데도 MCI 송신됨 → scrpgSucYn 게이트가 현재 송신 경로에 없음
- **확인 필요**: 
  1. 로컬에서 4필드 값 들어가는지 디버그 재확인 (이전에는 들어갔음)
  2. PL에게 scrpgSucYn 체크 위치 확인 (서비스단 vs 앞단)
- **배운 점**: 하드코딩 테스트 시 실제 흐름과 다를 수 있음. 게이트 조건이 어디에 걸려있는지 전체 경로 추적 필요

### [BRIDGE] 당행 보험계약대출 테스트 (NH농협생명)
- **상태**: 부분확인
- **결과**:
  - 서버단(fctcb1300tS): fncorgC="440" → getNhLifeUrl() 정상, NS1010_IR 응답 00000
  - 관리자페이지: fco1003/440/NH농협생명 확인
  - 프론트(bridge_common): MT 1초 리턴, PT 멈춤 — PL 배포 대기

---

## 2026-10-29

### [MCI] 사업자 4필드 하드코딩 테스트 결과
- **상태**: 부분해결
- **원인 확정**: getHometaxBzmnData(cmprRqSqno) 조회 결과 없음 — TB_EF_FC_CH_HTAX_PSN_BZMN에 10/23 이후 데이터 미적재 (스크래핑 미연결)
- **조치**: fctcb1100tS.java에서 hometaxMap에 4필드 직접 하드코딩 (DB 의존 우회)
- **결과**:
  - 송신: 4필드 모두 정상 포함됨 확인
  - 수신: 여전히 validation 오류 — 제휴사 결과값 미유입 (박명수 과장 확인, 이전과 동일)
- **대기**: 황인준 과장 스크래핑 진입점 연결 → 실데이터로 재테스트

### [BRIDGE] PT/MT 브릿지URL lnRqSqno 파라미터 추가
- **상태**: 부분확인
- **작업 내용**:
  - CM SQL(fcmab01_sql.xml): selectLoanExecProductDetail SELECT에 A.LN_RQ_SQNO 추가
  - CM 서비스(fcmab0106sS.java): dataMap.getOut().put("LN_RQ_SQNO", ...) 추가
  - PT JSP(fcmap0106r.jsp): lnRqSqno 변수 선언 + gotoInsurance params 추가
  - 상품리스트(fclmp0100r.jsp): gotoDtlProd에 lnRqSqno 파라미터 전달
- **테스트 결과**:
  - PT 데스크탑: 검정화면 멈춤 (opa.execute PC 미지원, 예상대로)
  - PT 실기기(올원뱅크 앱): 페이지 정상 이동 ✓
  - MT 데스크탑: brpgUrl 콘솔 정상 출력 ✓
  - MT 실기기(스마트뱅크 앱): 로그인 불가로 미확인

### 남은 할 일
- [ ] MT 실기기 테스트 (스마트뱅크 로그인 후)
- [ ] 155 DB 캡처 마무리
- [ ] 156 대출실행내역 수정 여부 양식 작성
- [ ] 157~159 보류 처리 팀장 보고
- [ ] 스크래핑 연결 후 MCI 4필드 실데이터 재테스트

---

## 2026-11-02

### [CBT] 제휴사 for문 변수 누적 버그 수정
- **상태**: 수정 완료
- **문제**: for문 밖에 선언된 변수들이 제휴사 반복마다 초기화 안 되어 이전 제휴사 데이터 누적
- **원인**: exist(Set), inList2, GdLIST, CarInfoLIST, carRgList, carInfList 등이 for문 밖 선언
- **해결**:
  1. `Set<String> exist` → for문 안으로 이동
  2. `inList2` → cbtOnly일 때 `else { inList2 = new ArrayList<>(); }` 추가
  3. `GdLIST`, `CarInfoLIST`, `carRgList`, `carInfList` → for문 밖 null, 안에서 `new ArrayList<>()` 재생성
- **검증**: cbtOnly 제휴사 → 일반 제휴사 순서로 테스트, 데이터 분리 확인
- **배운 점**: `.clear()`보다 `new ArrayList<>()`가 안전 — map101에 put한 리스트가 참조 공유로 같이 비워지는 것 방지

### [CBT] fisCodeList 제휴사 중복 이슈 클로즈
- **상태**: 이슈 아님 (기존 코드에서 처리됨)
- **내용**: cbtKey(AFLCN_C|AFLCN_WRS_C) 기반 Set<String> seen으로 중복체크 이미 구현
- **조치**: 이슈 클로즈

### [MCI] ntsBzcc SQL 오타 수정
- **상태**: 수정 완료
- **문제**: MCI 송신전문에 업종코드(722000) 대신 업종명(응용 소프트웨어 개발 및 공급업)이 전송됨
- **원인**: SQL alias가 `bts_bzcc`로 오타 → 업종명 컬럼을 참조
- **해결**: `bts_bzcc` → `nts_bzcc`로 수정

### [BRIDGE] brpgUrl APP/WEB 분기 추가 (PD003/PD301)
- **상태**: 수정 완료
- **문제**: PD301(사업자) 추가 시 "40000950"(앱용 wrsc코드)이 MT(웹)에도 내려가서 빈 페이지
- **해결** (fctcb1300tS.java):
  1. PD003 조건에 PD301 추가: `"PD003".equals(aflcnWrsC) || "PD301".equals(aflcnWrsC)`
  2. APP/WEB 분기: `!"05".equals(chnlCd)` → "40000950"(앱), else → `getNhbUrl("03","11")`(웹)
  3. `lnRqSqno` + `rq_gbn=02` 서비스단 블록 삭제 (440/449 전용이므로 서비스에서 일괄처리 불필요)
- **참고**: chnlCd="05"가 MT(모바일웹). yml 키는 J011.URL.03.11
- **배운 점**: "40000950"은 URL이 아니라 PT gotoNhwrc용 wrsc 서비스코드. MT는 yml URL 필요

### [MCI] NA1010_IS vs NA1010_IR 이슈 클로즈
- **상태**: 이슈 아님
- **내용**: send response이므로 IR이 맞음. 클로즈

### 이전 이슈 상태 업데이트
- [x] filterCarLoanOwners: dev 배포 완료 확인
- [x] fisCodeList 제휴사 중복: 기존 코드에서 Set seen으로 처리됨 (이슈 아님)
- [x] inList2 누적: if문으로 감싼 게 원인 → exist/inList2/리스트 초기화 수정 완료
- [x] NA1010_IS/IR: IR이 맞음 (클로즈)

### 남은 할 일
- [ ] MT 실기기 테스트 (스마트뱅크 로그인 후)
- [ ] 155 DB 캡처 마무리
- [ ] 156 대출실행내역 수정 여부 양식 작성
- [ ] 157~159 보류 처리 팀장 보고
- [ ] 스크래핑 연결 후 MCI 4필드 실데이터 재테스트
- [ ] 하드코딩 코드 전부 원복 (커밋 전)
- [ ] scrpgSucYn 게이트: 사업자 경로에서 if문 안에 있는 것 확인됨. 원복 후 dev 미배포 또는 값 출처 추적 필요

---

## 2026-11-03

### [환경] 통합테스트(통테) 기간 진입
- **상태**: 진행 중
- **테스트 환경**: smartdev 8700 (검증서버)
- **내용**: 단위테스트 → 통합테스트 단계 전환. smartdev 8700 서버에서 테스트 진행

### [CBT] 통합테스트 시나리오 실행 — 전체 PASS
- **상태**: 완료 (코드 문제 없음)
- **테스트 결과**:
  | 시나리오 | 조건 | 결과 | 판정 |
  |----------|------|------|------|
  | 1 | CBT 상품 2개 조회 | 2개 정상 조회 | PASS |
  | 2 | 거래제한 상품 외 다른 상품만 조회 | 1건 조회 | PASS |
  | 3 | 시나리오1과 동일, 중복 확인 | 중복 없음 | PASS |
  | 4 | 기존 로직 미변경 확인 | 정상 조회 | PASS |
  | 5 | 회귀테스트 | 통과 | PASS |

### [CBT] 버그 의심 → 데이터 문제로 클로즈
- **상태**: 이슈 아님 (데이터 문제)
- **증상**: 제휴사 거래제한 + CBT 등록 상품 1건인데 2건 조회됨
- **원인 조사**:
  1. ps325 SQL 분석 → 상품 단위 CBT 필터 정상 (line 283: `M.AFLCN_WRS_AMN_C = TA.AFLCN_WRS_AMN_C`)
  2. DB 조회 (`TB_EF_FC_CM_CBT_AMN_BSC WHERE AFLCN_C='fco1001'`) → **tpd1520, tpd1321 두 상품 모두 CBT 활성 등록 상태**
- **결론**: SQL/Java 코드 정상. 관리자페이지에서 두 상품 모두 CBT 등록되어 있어서 2건 조회는 정상 동작

### [MCI] 80002230 오류
- **상태**: 기록
- **내용**: 스크래핑여부(scrpgSucYn) 관련 오류코드

### [IQMS] 통합테스트 케이스 진행 현황
- **테스트 환경**: smartdev 8700 (검증서버)
- **진행 상태**:
  - TS-CP-0028 (가심사요청 IN): 005~008 — 진행 예정
  - TS-CP-0029 (결과통지 역거래): 001~004 — 진행 예정
  - TS-CP-0030 (대출신청접수): 001~002 — 진행 예정
  - TS-CP-0031 (실행결과통지): 001~007 — **내일부터 진행**
- **TS-CP-0031 주의사항**:
  - 001 (실행결과통지 전문 수신): 104번 역거래 수신 로그 확인 (MCI 관리자페이지 400번 로그)
  - 102번 = 가심사 결과통지 (한도조회 결과 → 상품 조회용)
  - 104번 = 대출실행 결과통지 (실제 대출 실행 결과 → 완료 처리용)
  - 102번과 104번 혼동 주의

### 남은 할 일
- [ ] TS-CP-0031-001~007 (실행결과통지) 통합테스트 진행
- [ ] 102번(가심사) vs 104번(실행결과) 역거래 구분 확인
- [ ] MT 실기기 테스트 (스마트뱅크 로그인 후)
- [ ] 스크래핑 연결 후 MCI 4필드 실데이터 재테스트
- [ ] 하드코딩 코드 전부 원복 (커밋 전)
- [ ] scrpgSucYn 게이트: 원복 후 dev 미배포 또는 값 출처 추적
- [ ] SourceTree MT merge 재시도 (STS 종료 후)

---

## 2026-10-30

### [CBT] 테스트 시나리오 실행 결과
- **상태**: 대부분 PASS, 버그 2건 발견

#### 테스트 결과
| 케이스 | 조건 | 예상 | 결과 | 판정 |
|--------|------|------|------|------|
| 1+4 | CBT전용 제휴사 + 거래제한 상품 + CBT등록 | CBT 상품 조회 | 정상 | PASS |
| 2+4 | 정식 제휴사 + 거래제한 + CBT | CBT 상품 1건 | 정상 | PASS |
| 3 | 회귀 테스트 (CBT 없는 기존 상품) | 상품 2개 정상 | 정상 | PASS |
| 5 | 정식출시 상품 2개 + 하나만 CBT등록 | 2개 모두 조회 | 2개 정상 | PASS |
| 6 | 정식출시 CBT 상품 보임 확인 | 보임 (ps315에서 조회) | 정상 | PASS |
| 추가 | 거래제한(CBT미등록) + 정상출시(CBT등록) | CBT건 1건만 | 1건 | PASS |

#### CBT 핵심 동작 원리 (테스트로 확정)
- **CBT는 ADDITIVE**: ps315가 모든 정식출시 상품을 리턴, ps325가 CBT전용 상품을 추가
- **ps325 NOT EXISTS = 중복제거용**: 정식출시 상품을 숨기는 게 아니라, ps315와 ps325 양쪽에서 나오는 걸 방지
- **거래제한 우회**: CBT에 등록된 상품만 해당. 미등록 상품은 거래제한 그대로 적용

### [CBT] 버그 1: filterCarLoanOwners HashMap 수정 미반영
- **상태**: PL 다음주 확인 예정
- **문제**: PL이 수정한 filterCarLoanOwners 코드(sendMap null 선언 → for문 안 new HashMap)가 dev에 배포 안 됨
- **증상**: sendInList2에 같은 상품이 중복 조회됨 (HashMap 참조 버그 — 루프 밖 선언으로 마지막 값만 반복)
- **확인 과정**:
  1. removeAflcnC 메서드 확인 → new HashMap이 루프 안에 있어 정상 (범인 아님)
  2. filterCarLoanOwners가 수정 전 코드로 돌고 있었음 확인
- **조치**: PL이 다음주 배포 확인

### [CBT] 버그 2: fisCodeList 제휴사 중복 + inList2 누적
- **상태**: PL 다음주 확인 예정
- **문제**: 
  1. ps310 + ps320 addAll 시 같은 제휴사가 양쪽에 있으면 fisCodeList에 2건 (제휴사 레벨 중복제거 없음)
  2. inList2가 제휴사 외부 for문 돌면서 초기화 안 되고 누적됨
- **증상**: fco1001로 조회했는데 fco0004 상품 3개가 같이 나옴 (inList2에 이전 제휴사 상품 잔류)
- **확인 과정**: fco0004 처리 시 inList2=3건 정상, fco1001 처리 시 inList2=5건(fco0004 3건 + fco1001 2건)
- **추가 참고**: PL이 송수신 테이블 확인 시 NA1010_IS로 봐야 하는데 NA1010_IR로 보내고 있어서 결과가 다를 수 있다고 함
- **예상 수정**:
  - 제휴사 중복: fisCodeList에 Set 기반 dedup 추가
  - inList2 누적: for문 시작 시 inList2 초기화 또는 루프 안 선언

### [CBT] ps325 OR 괄호 수정
- **상태**: 수정 완료 (기능 영향 없음)
- **내용**: 정식출시 상품제외 NOT EXISTS 내 OR 연산자 우선순위 버그
- **수정**: OR 양쪽 조건을 외부 괄호로 감싸서 AND와 결합 순서 보장
- **참고**: SQL 논리적으로는 맞는 수정이나, Java exist.add() 중복제거가 있어서 실제 출력은 변화 없음

### 남은 할 일 (다음주)
- [ ] PL 확인: filterCarLoanOwners 배포 상태
- [ ] PL 확인: inList2 누적 버그 (외부 for문 초기화)
- [ ] PL 확인: fisCodeList 제휴사 중복제거
- [ ] PL 확인: NA1010_IS vs NA1010_IR 송수신 방식
- [ ] MT 실기기 테스트 (스마트뱅크 로그인 후)
- [ ] 155 DB 캡처 마무리
- [ ] 156 대출실행내역 수정 여부 양식 작성
- [ ] 157~159 보류 처리 팀장 보고
- [ ] 스크래핑 연결 후 MCI 4필드 실데이터 재테스트

---

## 2026-10-27

*(기록 없음)*

<!-- 템플릿
### [카테고리] 제목
- **상태**: 해결 / 미해결 / 보류
- **문제**: 
- **원인**: 
- **해결**: 
- **배운 점**: 

카테고리: BUG, MCI, CBT, DB, CONFIG, TEST, BRIDGE, 기타
-->
