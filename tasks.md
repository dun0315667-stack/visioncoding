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
  2. DB 조회 (`TB_EF_FC_CM_CBT_AMN_BSC WHERE AFLCN_C='fco1001'`) → **fpd1320, fpd1321 두 상품 모두 CBT 활성 등록 상태**
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

### [IQMS] TS-CP-0031 통합테스트 진행
- **상태**: 진행 중
- 001 신용대출실행결과통지 전문 수신 — 완료 (100번 로그에서 금리/한도 필드 확인)
- 005 응답 전문 송신 확인 — 진행 중 (400번 로그에서 조회결과_OUT, 응답코드, 응답메세지 확인)
- 007 오류 처리 여부 — 양식 작성 (fctcb1200tS 오류 처리 소스코드 캡처)
- **MCI 로그 구분**: 100번=수신 데이터(001 증빙), 400번=응답 송신(005 증빙)

### [BRIDGE] 440/449 gotoInsurance 파라미터 확인
- **상태**: 확인 완료
- 올원앱에서 NH생명 화면 이동 정상, DB에 yml URL 정상 저장
- lnRqSqno, rq_gbn='02'는 네이티브 브릿지로 전달 → 수신 확인은 제휴사(NH생명) 쪽 영역
- 요구사항 정의서(ABFBP0001R, svcID/prod/lnRqSqno/rq_gbn) 매칭 확인 완료

### [인프라] ODP 서버 연결 장애
- **상태**: PL 보고 완료, 복구 대기
- **증상**: 16.24.117.96:7791 Connection refused — 로컬/배포서버/모바일 전부 안 됨
- **원인**: ODP(Online Data Platform) 서버 문제 (PL 확인)
- **참고**: API Gateway가 아니라 ODP임

### [CBT] ps320 NOT EXISTS 로직 분석
- **상태**: 분석 완료
- ps320 주석: "정식출시돼서 원본에도 이미 노출되는 상품은 CBT쪽에서 제외(중복 방지)"
- 요구조건: "지정된 이용자에 한해 **거래제한 상태**의 기관/상품의 거래가 가능하도록 허용"
- ps320/ps325 NOT EXISTS = 정식출시 상품 자동 제외 (요구조건에 맞게 동작)
- **정식출시 상품 + CBT등록 고객 → 2건 조회**: 정상 (ps315에서 2건, ps325에서 0건. CBT는 상품을 숨기는 게 아니라 거래제한 상품을 추가 허용하는 기능)
- **담당 범위**: ps320(CBT 제휴사), ps325(CBT 상품)

### 남은 할 일
- [ ] TS-CP-0031 나머지 케이스(002~004, 006) 진행
- [ ] ODP 복구 후 로컬 테스트 재개
- [ ] 배포리스트 엑셀 작성
- [ ] MT 실기기 테스트 (스마트뱅크 로그인 후)
- [ ] 스크래핑 연결 후 MCI 4필드 실데이터 재테스트
- [ ] 하드코딩 코드 전부 원복 (커밋 전)
- [ ] scrpgSucYn 게이트: 원복 후 dev 미배포 또는 값 출처 추적
- [ ] SourceTree MT merge 재시도 (STS 종료 후)

---

## 2026-11-05

### [MCI] 로컬 더미로그인 사업자 테스트 불가 이슈
- **상태**: 조사중
- **증상**: 로컬 더미로그인으로 MCI 가심사요청 테스트 시 jonCls, lnRqJobC 빈값 → 사업자/직장인 분기 미진입 → MCI 전문 미송신
- **원인**: 더미로그인이 실제 고객정보 조회를 안 거쳐서 직업코드(lnRqJobC)가 세팅 안 됨
- **시도한 것**:
  1. fctcb1100tS에서 jonCls="5" 강제 세팅 → 효과 없음 (CM/CH에서 이미 빈값으로 처리됨)
  2. 세션로그인 코드에서 lnRqJobC="5" 강제 세팅 → 효과 없음 (맵에 다시 put 안 됐을 가능성, 또는 jonCls가 별도 경로로 세팅)
- **배포서버에서는**: 실 인증 거치므로 정상 동작 (전문 나감, 3필드 빈값은 스크래핑 미연결 때문)
- **결론**: 로컬 디버깅은 보류, 배포서버 기준으로 테스트 진행

### [배포리스트] PT 배포리스트 엑셀 작성
- **상태**: 진행 중
- **작업**: CH/CM/TI/PT/MT 각 저장소의 삭제/변경/신규추가 시트에서 프로그램기능 빈칸 채우기
- **변경 시트**: 프로그램기능 + 사유까지 작성 필요
- **참고**: AM 배포리스트 양식 기준

### [학습] 우선 학습 과제 1 — 데이터 흐름 추적 (jonCls/lnRqJobC)
- **우선순위**: 높음
- **주제**: Spring + 프로프레임웍에서 requestMap 데이터가 CM → TI 계층을 거치며 어떻게 전달되는지 추적하는 방법
- **핵심 발견**: lnRqJobC는 CM에만, jonCls는 TI에만 존재. 중간 변환 지점을 못 찾음
- **TI에서 jonCls 가져오는 코드**: `String jonCls = TIUtil.safeTrim(mapIn, "jonCls")`
- **CM에서 lnRqJobC 가져오는 코드**: `String lnRqJobC = dataMap.getin.getValue("ln_rq_job_c", "")`
- **구체적으로 모르는 것**:
  1. lnRqJobC가 최초로 세팅되는 위치 (세션로그인? 고객정보 조회? CM 컨트롤러?)
  2. lnRqJobC → jonCls로 변환되는 지점 (CM→TI 사이 어딘가)
  3. mapIn에 jonCls 키로 언제 put되는지
  4. 더미로그인 vs 실 로그인에서 고객정보 세팅 차이
- **학습 방법**:
  1. STS에서 lnRqJobC `Ctrl+H` 전체 검색 → 세팅하는 곳 전부 목록화
  2. jonCls도 동일하게 전체 검색 → 세팅 위치 파악
  3. `mapIn.keySet()` 콘솔 출력으로 TI에 어떤 키가 넘어오는지 확인
  4. 브레이크포인트로 CM부터 TI까지 한 단계씩 따라가기
- **목표**: "이 값이 어디서 세팅되고 어디서 꺼내지는지" 전체 경로를 스스로 추적할 수 있는 능력

### [학습] 우선 학습 과제 2 — 프로젝트 환경 설정 파일 이해
- **우선순위**: 중간
- **배경**: 배포리스트 작성하면서 파일 기능을 모르는 것들이 많았음
- **학습 대상**:
  1. **빌드 시스템 (Gradle)**:
     - build.gradle — 프로젝트 빌드 설정 (의존성, 플러그인, 태스크)
     - dependencies-app/common/system.gradle — 의존성 분리 관리
     - settings.gradle — 멀티모듈 프로젝트 설정
  2. **CI/CD (GitLab)**:
     - gitlab-ci*.yml — 빌드/테스트/배포 파이프라인 정의
     - variables-*.yml — 환경별(dev/predev/run) 변수 설정
     - 환경별 분리: dev=개발, predev=스마트데브, run=운영, dr=재해복구
  3. **Spring Boot 설정**:
     - application.yml — 기본 설정
     - application-local.yml — 로컬환경 설정 (커밋 금지)
     - application-cbt.yml — CBT 환경 설정
     - MtApplication.java / PtConfiguration.java — 스프링 부트 진입점/설정 클래스
     - WebSecurityConfig.java — 웹 보안 설정 (CORS, CSRF, 인증 필터)
  4. **보안/암호화**:
     - ARIA (ARIAEngine/ARIAUtil) — 농협 표준 암호화 알고리즘
     - CipherUtil/CryptoUtil — 암복호화 유틸
     - eziok* — 본인인증 라이브러리 (간편인증)
     - PETRA — 농협 표준 DB 암호화 (dev서버 자동동작)
  5. **연계 시스템**:
     - ODP (Online Data Platform) — 농협 내부 데이터 수집 미들웨어 (Od~ 접두어)
     - 공공마이데이터 — 건강보험/홈택스/나이스 등 외부기관 연계 (Pm~ 접두어)
     - MCI — 제휴사 연계 전문 통신
  6. **배포 관련**:
     - manifest_dev/dr/run.yml — Cloud Foundry 배포 매니페스트
     - dev.sh/dr.sh/run.sh — 환경별 실행 스크립트
     - nhcommonab-*.exe — 정적 리소스 빌드 도구
  7. **패키지 구조 (채널팀)**:
     - ag=약관, co=공통, cr=인증, ji=직장, lm=배너, ma=메인/상품, ml=관리자(BT), ti=TI연계, ws=웹서비스, bn=배너
     - 접미어: sC=컨트롤러, sS=서비스, D=Dao, tC=TI컨트롤러, tS=TI서비스
     - PT=고객화면, BT=관리자화면, 공통=co패키지

---

## 2026-11-09

### [TEST] 테스트 커버리지 첫 작업
- **상태**: 진행 중
- **환경**:
  - 포탈: https://16.24.117.168:28080/module (테스트 커버리지 메뉴)
  - 측정 방식: dev 서버에서 시나리오 실행 → 포탈에서 자동 수집 (실시간)
  - "탔다" = 해당 코드 라인이 실행됨 = 커버리지 초록으로 변경
- **작업 흐름 (파악 완료)**:
  1. 포탈에서 파일 검색 → 라인 커버리지 % 확인
  2. 빨간(미실행) 메서드 확인 → 어떤 시나리오가 타는지 파악
  3. dev 서버에서 해당 시나리오 실행 (PT 28800=PC, MT 28700=폰)
  4. 포탈에서 커버리지 변화 확인
- **결과**:
  - fcccb1300tS (대출신청접수): reqLoanExec **46% → 80%+** (dev 8700 TI로 실행)
  - fcmam0106sC (상품상세 컨트롤러): 28.26% → 일부 상승

#### 못 태운 메서드 (사유)
| 파일 | 메서드 | 사유 |
|------|--------|------|
| fcccb1300tS | updtLoanData | 역거래(104번) 수신 필요 — 외부 의존 |
| fcccb1300tS | setCoocon106Api | 쿠콘 외부 API 연계 필요 |
| fcccb1300tS | reqLoanCancel | 대출 취소 시나리오 필요 |
| fcmam0106sC | productDetailViewMo | 비교신청일련번호 비어있는 진입경로 없음 |
| fcmam0106sC | getMortgageCnt | 주담대(CH) 관련 — 비교대출 흐름에서 안 탐 |

#### 배운 것
- **View vs ViewMo**: View=PC(PT), ViewMo=모바일(MT). PT에서 실행하면 View만 탐
- **JSP↔Controller 연결 찾기**: 컨트롤러명으로 JSP 검색 (Ctrl+H → `*.jsp, *.js`), `fg.sf.exec("서비스ID")` 패턴
- **resultMap 리턴 = JSON 응답** (JSP 뷰 이동이 아님, Ajax 콜백)
- **에러도 커버리지**: 에러 분기 코드(catch/throw)가 실행되면 해당 라인 초록 전환
- **더미 아이디 주의**: 아이디에 따라 PETRA 복호화 에러(FCT000002) 발생 가능 — 되는 아이디로 작업할 것

#### IQMS 통합테스트 보류 처리
- UI 이미지 없는 건 전부 보류 처리 (PL 지시)
- TS-CP-0031-007 (대출실행수신 오류처리) 양식 작성 완료

### [학습] 우선 학습 과제 3 — 테스트 커버리지 실무
- **우선순위**: 높음
- **배경**: 첫 커버리지 작업에서 코드 추적/시나리오 매핑 역량 부족 체감
- **학습 대상**:
  1. **컨트롤러→JSP 매핑 추적**:
     - @RequestMapping / 서비스ID 기반 매핑 이해
     - fg.sf.exec("서비스ID") → Controller 연결 구조
     - yml 설정 vs 어노테이션 매핑 차이
  2. **메서드 호출 체인 추적 (STS)**:
     - Ctrl+Alt+H (Call Hierarchy) — 누가 이 메서드를 호출하는가
     - Ctrl+Shift+G (References) — 이 메서드가 어디서 참조되는가
     - Ctrl+H (File Search) — 파일/텍스트 전체 검색
  3. **분기 조건 파악**:
     - if/else 분기에서 어떤 값이 어떤 경로를 타는지 추적
     - 파라미터 origin 추적: 값이 세션? 요청? DB? 어디서 오는가
     - wrsTrDsc, chnlCd, jonCls 같은 코드값의 의미 파악
  4. **시나리오↔코드 매핑**:
     - "이 화면에서 이 버튼 누르면 어떤 메서드가 실행되는가" 역추적
     - "이 메서드를 태우려면 어떤 화면 시나리오가 필요한가" 정추적
  5. **커버리지 도구 활용**:
     - 포탈 화면 구조: 파일 검색, 메서드별 커버리지, 라인별 색상
     - 빨간/초록/노란 라인 의미
     - 측정 서버 확인 및 재측정 방법
- **학습 방법**:
  1. 내가 수정한 파일(fctcb1100tS, fctcb1101tS 등)부터 Call Hierarchy 전부 따라가기
  2. 각 메서드의 호출 경로를 종이/메모에 그려보기 (Controller → Service → Dao)
  3. dev에서 시나리오 실행 후 "어떤 코드가 탔는지" 포탈에서 확인하면서 감 잡기
- **목표**: "빨간 메서드 → 어떤 시나리오 필요한지 → 실행 → 초록" 사이클을 혼자 돌릴 수 있는 수준

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
