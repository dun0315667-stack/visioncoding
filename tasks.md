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

---

## 2026-11-10

### [접근성] 접근성심사(웹 접근성 진단) 대응 — MT 모바일웹
- **상태**: 진행 중
- **대상**: 비교대출 모바일웹(MT) smartdev 8700 기준
- **심사 범위**: 모바일웹 / iOS / Android 총 3건. iOS/Android는 PT 쪽 담당
- **수정 원칙**:
  1. 진단결과서에 지적된 항목만 수정, 범위 밖 수정 금지
  2. 공통 함수(nhui.toggleSlide, nhui.toggleKeyword 등) 건드리지 않기
  3. CSS 수정은 퍼블리셔 담당으로 패스

#### 수정 완료 파일 및 내용

**fccrm0400r.jsp (본인인증 화면)**
- title 속성 정리, aria-label 추가

**fcagm0102p.jsp (약관동의 팝업)**
- 92/98/101번 줄 `.attr("title", "상세 동의 팝업 열기 선택안됨/선택됨")` → `"약관 상세보기 선택안됨/선택됨"`으로 변경
- initPage() 함수 내 셀렉터 확인: `span[id^='describedby1']` (id 기반, title 아님 → title 변경 안전)

**fcagm0100r.jsp (약관동의 화면)**
- renderDivFormAccordionDom(ag) 함수: 하드코딩 aria-label → `escHtml(ag.TINM) + ' 펼치기/접기'`로 동적 변경
- 403번 줄 tempHtml의 button에 동적 aria-label 적용

**UI-MWMC-10100_F.html (약관동의 모달 팝업)**
- 검사항목 4-1: 닫기 버튼 순서 변경 (마지막으로 이동)
- 30번 줄 title="" 빈값 → "약관동의 팝업 닫기"

**fclmm0100r.jsp (상품 목록 화면)**
- "상품보기" 버튼에 상품명 추가: `aria-label="${lowBasWsr.WRS_HGNM} 상품보기"`
- 키워드 필터 버튼: title="" → "키워드 필터 펼치기", blind 텍스트 구체화

**fcjim1400r.jsp (추가정보 입력)**
- 검사항목 6-2 (색상만으로 상태 구분): CSS 수정 필요 → 퍼블 담당으로 패스
- 차량번호 input vhrNo의 aria-label="123서1234" → "내 차량 번호 입력"으로 변경

#### 기술 학습 사항
- **title 속성 vs JS 셀렉터**: `.attr("title", ...)` = 값 세팅(set), `$("[title='...']")` = 셀렉팅(select). 셀렉팅하지 않으면 title 값 변경 안전
- **동적 HTML에서 aria-label**: 함수 파라미터(ag.TINM 등)를 escHtml()로 XSS 방지 후 aria-label에 삽입
- **PT/MT/앱 범위 구분**: 접근성심사가 MT 기준 → MT 코드만 수정. PT는 iOS/Android 심사에서 별도 대응
- **퍼블 파일**: UI-MWMC-XXXXX.html = 퍼블리셔가 만든 HTML 마크업, JSP가 이 구조 참고

### [접근성] 퍼블리셔 미팅 — 추가 수정 사항 정리
- **상태**: 보류
- **내용**: 포커스 순서 변경, 선택됨/미선택 title, 미승인내역 a→div 변환 등
- **대기**: 퍼블리셔 파일 참고해서 추가 작업 예정

### [CONFIG] ti_config.yml 당행상품이동 모바일브릿지 URL 추가
- **상태**: 부분 완료
- **완료**: local/dev/predev 3개 환경에 `URL: "03": "11": "URL"` 추가
- **대기**: 운영환경(dr/run) URL 미정 → 팀장에게 URL 받은 후 추가

### [EFFCCOM] git 충돌 해결
- **상태**: 해결
- **문제**: pull 시 `MERGE_HEAD exists` 에러 + 2개 파일 충돌
- **해결**: `git merge --abort` → `git reset --hard origin/local` → pull → 충돌 2건 "상대 것으로 해결"
- **주의**: EFFCCOM은 내 코드가 아님. 커밋하면 안 됨, 충돌 시 항상 원격 기준으로 해결

### [기타] .gitignore 실수 복구
- **상태**: 해결
- **문제**: 접근성 수정 JSP 커밋 시 .gitignore가 같이 스테이지됨
- **해결**: 커밋 되돌리기 → JSP 파일만 선택해서 재커밋
- **배운 점**: 커밋 전 스테이지 목록 반드시 확인. 의도하지 않은 파일이 올라가지 않았는지 체크

### [학습] 팀장 다운로드 코드 패턴 — study-practice.md 세션 7 추가
- **상태**: 완료
- **내용**: fcmam0100r.jsp의 mvDownLoad 함수 분석 → study-practice.md에 세션 7로 추가
- **핵심 패턴**: yml → SpringContext.getProperty → JSP `<c:set>` → JS `'${변수}'` → OS/앱타입 분기 → openWebPage
- **비교**: getNhbUrl (TI config yml 계층), getEffcCore (TI 서비스 yml 매핑)과의 차이

### 남은 할 일
- [x] 퍼블리셔 파일 참고 추가 접근성 수정 (포커스 순서, 선택됨 title 등) → 11/11 대부분 완료
- [ ] ti_config.yml 운영환경(dr/run) URL 추가
- [ ] TS-CP-0031 나머지 케이스(002~004, 006) 진행
- [ ] 배포리스트 엑셀 작성
- [ ] MT 실기기 테스트
- [ ] 스크래핑 연결 후 MCI 4필드 실데이터 재테스트
- [ ] 하드코딩 코드 전부 원복 (커밋 전)

---

## 2026-11-11

### [접근성] MT 접근성심사 추가 수정 — 퍼블 피드백 반영
- **상태**: 완료
- **수정 내용**:
  1. **fcagm0102p.jsp (약관동의 팝업, MT)**: `.attr("title", "상세 동의 팝업 열기 선택됨/선택안됨")` 4줄 전체 삭제 (퍼블 지시 — 불필요한 텍스트 정보 제거)
  2. **fcagm0100r.jsp (약관동의 화면, MT)**: renderDivFormAccordionDom 함수에서 aria-label 삭제 → `aria-describedby="agChk" + ag.STLT_EXPS_SQNO` + `<span class="blind">` + `escHtml(ag.TINM)` 추가
  3. **선형화 순서 (util-box)**: 닫기/다운로드 버튼 DOM 순서 swap (CSS flex-direction: row-reverse로 시각적 순서 유지)
  4. **미승인내역 displayHtml**: 대출불가 상품의 `<a href="javascript:void(0);" role="button">` → `<div>`로 변환
- **트러블슈팅**:
  - aria-describedby 수정 후 내용 안 나옴 → 원인: `sqno`는 agInputDom 내부 변수, renderDivFormAccordionDom에서는 `ag.STLT_EXPS_SQNO` 사용해야 함
  - blind span 추가 시 aria-describedby와 중복 읽기 가능성 → 퍼블 지시대로 둘 다 적용
- **배운 점**:
  - `aria-label` = 텍스트 직접 삽입, `aria-describedby` = 다른 요소 id 참조
  - `<span class="blind">` = 시각적으로 숨기되 스크린리더가 읽는 공통 CSS 클래스
  - 함수 간 변수 스코프 주의: 다른 함수의 지역변수를 참조하면 undefined 에러 → 렌더링 전체 멈춤

### [접근성] PT 접근성심사 대응 — title 삭제 + 포커스 이동
- **상태**: 진행 중
- **수정 내용**:
  1. **fcagp0100r.jsp (약관동의, PT)**: setChecked 함수 내 `.attr('title', isOn ? '상세 동의 팝업 열기 선택됨' : '선택안됨')` 2줄 삭제 (MT와 동일 지적)
  2. **fcmap3100r.jsp (PT 메인)**: 금융사선택 버튼 2개(line 69, 124) 삭제 — 퍼블 HTML(UI-MWMC-70000)에 없음, 접근성 진단서 PT 지적
  3. **fcmap0100r.jsp (공유하기 팝업, PT)**: 포커스 이동 구현
     - FCCOP0120P.jsp: `<div class="modal-window">` → `<div class="modal-window" tabindex="-1">`
     - fcmap0100r.jsp: `fcPop.modalOpen('FCCOP0120P')` 콜백에 `setTimeout` + `.focus()` 추가
     - 셀렉터 확정 과정: `#FCCOP0120Pmodal .modal-window` → null / `.modal-window` → 3개 / `.slide .modal-window` → 공유하기 팝업만 특정됨
- **셀렉터 확정 방법론**:
  1. F12 Elements breadcrumb에서 부모→자식 구조 파악
  2. 콘솔에서 `document.querySelector('셀렉터')` 로 요소 리턴 확인
  3. `querySelectorAll('셀렉터').length` 로 고유성 검증 (1이어야 안전)
  4. 3 이상이면 부모 클래스(`.slide`)를 조합해서 좁히기
- **트러블슈팅**: `function() => {}` 문법 에러 → 파일 전체 파싱 실패 → productDetail ReferenceError → 페이지 멈춤. `function() {}`과 `() => {}` 절대 섞지 말 것
- **적용 완료**: 문법 수정 후 포커스 이동 정상 동작 확인

### [MCI] 사업자 업종코드 분기 설계 — PL 회의
- **상태**: 설계 완료, 구현 예정
- **배경**: MCI 사업자 가심사요청 시 4필드 중 국세청업종코드(nts_pnp_bzcc)가 빈값으로 나감
- **PL 지시**: 스크래핑 Y/N을 "보내냐 마냐"가 아니라 "어디서 값을 가져오냐"의 분기로 변경
- **설계**:
  - 기존 ps111 쿼리(fctcb01_sql.xml)에 LEFT OUTER JOIN 이미 있음 (홈텍스 H + 비교신청관리 R)
  - SELECT 컬럼만 수정:
    - `NVL(H.COMM_BZMN_YN, 'N')` — 공동사업자여부
    - `CASE WHEN H.NTS_BZCC IS NOT NULL THEN H.NTS_BZCC ELSE CASE R.BZMN_BZCC WHEN '1' THEN '369908' ... END END` — 업종코드
    - `NVL(H.DLQ_YN, 'N')` — 체납여부
  - Java if/else 분기 안 건드림, SQL에서 NULL 처리
- **업종코드 매핑** (비교신청관리 BZMN_BZCC → NH API 코드):
  | BZMN_BZCC | 분류 | NH API 코드 |
  |-----------|------|------------|
  | 1 | 제조업 | 369908 |
  | 2 | 도소매업 | 523999 |
  | 3 | 운수업 | 630909 |
  | 4 | 음식/숙박/교육/기타 | 930925 |
- **확인 필요**: BZMN_BZCC가 NULL인 경우(스크래핑 실패 + 직접입력 안 함) 기본값 처리 여부 → PL 확인
- **의사결정 (11/11)**: comm_bzmn_yn(공동사업자여부), io_ptax_dlq_yn(체납여부) 두 필드는 공백으로 보내기로 결정. 황인준 과장이 제휴사에 공백 수신 시 응답 문제 없는지 확인 중
- **의사결정 재검토 (11/12)**: 제휴사(김성현) 피드백 — 빈값 들어오면 해당 조건 자동승인 처리됨. 이게 맞는지 사업부서 확인 필요
  - **기존 플랫폼 현황**:
    - 카카오: 스크래핑 실패 시 고객 직접입력 → 체납여부/공동사업자여부도 선택 받음
    - 네이버/토스: 스크래핑 실패 시 거절 → 정상건만 값 제공
  - 황인준: 우리도 화면에서 직접입력 받도록 검토 중. 실패 시 아예 전문 안 보내고 있음
  - **영향**: "공백 전송" 의사결정이 변경될 수 있음. 방향에 따라 ps111 쿼리 설계 달라짐:
    1. 빈값 유지 → 현재 설계대로
    2. 직접입력 받기 → 업종코드처럼 CASE WHEN 추가 (체납/공동사업자도)
    3. 실패 시 전문 안 보내기 → 스크래핑 성공 여부 게이트 추가
  - **대기**: PL에게 방향 확인 필요

### [MCI] PT 더미로그인 사업자 테스트 제약
- **상태**: 조사 완료
- **증상**: 더미로그인 + 본인인증(실 주민번호) 해도 사업자 MCI 안 나감
- **원인**:
  1. 더미 고객번호에 사업자 데이터 없음 → jonCls≠"5" → 사업자 분기 미진입
  2. 스크래핑 스킵해도 jonCls 문제는 해결 안 됨 (첫 번째 관문)
  3. 주민번호 암호화: `encrytedRlno` 하드코딩해도 `setApiBody`의 74번 줄 `reqMap.put("cusPwizeRlno", tiUtil.safeTrim(map101, "RRNo"))` 가 별도로 덮어씀
- **결론**: 로컬/더미로그인으로는 사업자 MCI 전체 흐름 테스트 불가. dev 배포 + 실 인증 기준 테스트 필요
- **배운 점**:
  - MCI 전문은 `setMciBody`(바디)와 `setApiBody`(공통헤더) 두 곳에서 필드 세팅 — 하나만 수정하면 다른 쪽이 덮어쓸 수 있음
  - `pwize_tr_rlno`(바디)와 `cusPwizeRlno`=`RRNo`(헤더)는 서로 다른 필드
  - 더미 로그인 = 고객번호 기반 데이터 없음. 본인인증(주민번호)과 고객번호는 별개 개념

---

## 2026-11-12

### [접근성] PT 팝업 포커스 이동 — 추가 적용
- **상태**: 진행 중
- **수정 내용**:
  1. **FCAGP0101P.jsp (약관동의 팝업, PT)**: `modal-window floating`에 `tabindex="-1"` + `initPage`에 `setTimeout` + `focus()` 추가. 셀렉터: `.modal-window.floating`. `document.activeElement` 확인 완료
  2. **FCCOP1500P.jsp (통신사 선택 팝업, PT)**: `initPage` 함수 새로 생성. `.modal-window` 2개 존재 → `.slide .modal-window` 셀렉터로 특정. `setAttribute('tabindex', '-1')` JS에서 동적 처리 (JSP HTML 변경이 서버에 안 올라가는 문제 우회)
  3. **FCCOP0120P.jsp (공유하기 팝업, PT)**: 11/11 완료
- **학습 포인트**:
  - `document.activeElement`로 현재 포커스 위치 확인
  - `querySelectorAll('.modal-window').length`로 동일 클래스 요소 수 확인 → 2개 이상이면 셀렉터 좁히기
  - JSP HTML 수정이 서버에 반영 안 될 때 → JS `setAttribute`로 동적 처리 가능
  - `aria-hidden="true"` 경고: 팝업 열 때 뒤쪽에 자동 설정됨 → 포커스를 팝업으로 옮기면 해결
- **남은 접근성 항목**:
  - fcjim1400r.jsp PT: 차량번호 input aria-label 추가 (검출내용2)
  - fcjim1400r.jsp PT: 색상만으로 구분 → 퍼블 담당 (검출내용1)

### [MCI] 4필드 빈값 처리 — 제휴사 피드백 (의사결정 재검토)
- **상태**: 대기
- **내용**: 제휴사(김성현) 피드백 — 빈값 자동승인 문제. 카카오는 직접입력, 네이버/토스는 실패시 거절
- **스크래핑 체크 위치 논의**:
  - PL 언급: "CBT에서 했던 것처럼 list에서 for문 안에서 5개 돌면 하나 빼서 4개 돌게 하거나 continue 방식"
  - **for문 = 스크래핑 채널(카카오, 네이버, 토스, 페이코, 뱅크샐러드) 루프** (제휴사 for문 아님)
  - for문 전 체크: 스크래핑 전체 실패 → MCI 자체 안 보냄 (all or nothing)
  - for문 안 체크: 채널별로 성공/실패 구분 → 실패 채널만 continue(skip), 성공 채널 데이터만 살림 (partial success)
  - **유력 방향**: "스크래핑 실패 시 심사 전문 호출 안 함" (김성현 선배)
  - **미확정**: 전부 실패 시 MCI 자체 미전송인지, 성공 채널만 보내는지
- **대기**: PL에게 방향 확인 필요

### [접근성] PT 팝업 포커스 이동 — 입사일/연소득/면적선택
- **상태**: 진행 중
- **수정 내용**:
  1. **입사일 선택 팝업 (FCCOP1300P 추정)**: 기존 `initPage()` 함수에 `setTimeout` + `.modal-window` focus 추가. 기존 `focusOn()` 함수는 `.active` 버튼 scrollIntoView용으로 접근성 포커스와 무관
  2. **연소득 입력 팝업**: class="modal-window keypad-pop". 셀렉터: `.modal-window.keypad-pop` (고유하므로 `.slide` 불필요). initPage 내 setTimeout + tabindex + focus 추가
  3. **면적 선택 팝업 (FCMIP2201P, 주담대)**: 셀렉터 확정 `#FCMIP2201P .modal-window`. `querySelectorAll('.modal-window')` length=2, [1]이 해당 팝업. parentElement = `div#FCMIP2201P.modal.slide`
- **트러블슈팅**:
  - 입사일: initPage 코드 추가 시 오타 → SyntaxError → 팝업 바디 비워짐. 오타 수정 후 해결
  - 면적선택: `#FCMIP2201Pmodal` → null (id가 `FCMIP2201P`임). `modal-windnow` 오타 → TypeError null. tabindex 미설정 상태에서 focus() → activeElement가 li(뒤 요소)로 잡힘. setAttribute('tabindex', '-1') 선행 필요
- **면적선택 미완료**: tabindex + focus 패턴 확정됨, 실제 JSP 코드 적용 아직 안 됨

### [MCI] 1:N 테스트 — 관리자페이지 결과통지 확인
- **상태**: 확인 중
- **배경**: PL 주간회의 후 지시 — "당행 MCI 1:N 경우 생보/손보로 상품 결과통지 확인"
- **테스트 조건**: fco1003(계약대출) 상품, 440(NH생명) + 449(NH손보) 대상
- **1:N 구조**: fcccb1100tS에서 fisCodeList for문이 제휴사별로 각각 MCI 송신. 하나의 cmprRqSqno에 여러 lnWrsC(상품)
- **확인한 것 (440 NH생명)**:
  - 관리자페이지: fco1003 / 440 / NH농협생명 / NA1010_IR
  - 등록프로그램: DCB1200U (fctcb1200tS = 결과통지 역거래 수신)
  - 수신결과: 00000 (정상)
  - 송신전문: InWrsC="NHLIFE01", InLntAmt="23060000"
- **남은 확인**: 449(NH손보) 건도 동일하게 정상 수신인지 확인
- **확인 포인트 정리**:
  1. MCI 로그사이트: 같은 cmprRqSqno로 100번 송신 N건 확인
  2. MCI 관리자페이지: 400번 결과통지 제휴사별 각각 수신 확인
  3. 송수신 테이블(TB_EF_FC_CH_CNTGRM_SNR): cmprRqSqno + aflcn_c로 건수 확인

### [BRIDGE] brpgUrl 쿼리파라미터 분석 — 올원앱 홈 이동 이슈
- **상태**: 조사 필요
- **발견 사항**:
  1. DB(TB_EF_FC_CH_LN_EXEAMN)에서 brpgUrl — 10/29에는 쿼리파람 붙어있었고, 11/03부터 안 붙음
  2. **결론**: 파람은 PT JSP(fcmap0106r.jsp) `gotoInsurance` 함수에서 클라이언트단(JS)으로 붙이는 것 → DB에는 순수 URL만 저장이 정상. 10/29 파람 붙은 게 오히려 버그
  3. DB에 파람이 안 붙는 것은 코드상 정상 — fctcb1300tS.java에서 brpgUrl에 파람 append 코드 없음
- **새 이슈: 올원앱에서 홈으로 이동**:
  - 이전: Java에서 brpgUrl에 파람 강제 추가했을 때 → 제휴사 상품 페이지로 정상 이동
  - 현재: Java에서 파람 안 붙이니 → 제휴사 홈으로만 이동
  - **의미**: 제휴사 사이트가 쿼리파람(lnRqSqno, rq_gbn)으로 신청 건을 식별함. 파람 없으면 홈으로 보냄
  - **핵심 질문**: 올원앱에서 브릿지URL 이동 시 호출되는 URL = brpgUrl 값 그대로. gotoInsurance가 파람을 안 붙이고 있음
- **내일 확인할 것**:
  1. 올원앱에서 제휴사 이동 시 `gotoInsurance` 함수를 **타는지** 확인 (alert/브레이크포인트)
  2. gotoInsurance를 안 탄다면 → 실제 호출 경로 추적 (어떤 JS 함수가 brpgUrl로 이동하는지)
  3. gotoInsurance를 타는데 파람이 안 붙는다면 → 해당 함수 내부 로직 디버깅
  4. **PL에게 확인**: brpgUrl에 lnRqSqno 파람을 서버에서 붙여야 하는지 vs 클라이언트에서 붙여야 하는지
- **증빙 수집 (brpgUrl이 DB에 순수 URL만 저장됨을 증명)**:
  1. fctcb1300tS.java — brpgUrl 세팅 코드 캡처 (파람 append 없음)
  2. TB_EF_FC_CH_LN_EXEAMN SELECT — 11/03 이후 brpgUrl에 파람 없음
  3. fcmap0106r.jsp gotoInsurance — JS에서 클라이언트단 파람 추가 코드

### 남은 할 일
- [ ] **[긴급] 올원앱 brpgUrl 홈 이동 이슈** — gotoInsurance 함수 디버깅 + PL 확인
- [ ] **[긴급] MCI 1:N** 449(NH손보) 결과통지 확인 + 로그사이트/관리자페이지 캡처
- [ ] 면적 선택 팝업 (FCMIP2201P) JSP 코드에 tabindex + focus 적용
- [ ] ps111 쿼리 업종코드 CASE WHEN 수정 (fctcb01_sql.xml) — 4필드 방향 확정 후
- [ ] BZMN_BZCC NULL 처리 방침 PL 확인
- [x] PT 공유하기 팝업 포커스 이동 완료 (fcmap0100r.jsp + FCCOP0120P.jsp)
- [x] PT 약관동의 팝업 포커스 이동 완료 (FCAGP0101P.jsp)
- [x] PT 통신사 선택 팝업 포커스 이동 완료 (FCCOP1500P.jsp)
- [ ] 퍼블에게 MT common.css 업데이트 요청 (flex-direction: row-reverse)
- [ ] fcjim1400r.jsp PT: 차량번호 input aria-label 추가
- [ ] ti_config.yml 운영환경(dr/run) URL 추가
- [ ] 배포리스트 엑셀 작성
- [ ] MT 실기기 테스트
- [ ] 스크래핑 연결 후 MCI 4필드 실데이터 재테스트
- [ ] 하드코딩 코드 전부 원복 (커밋 전)

---

## 2026-11-20

### [MCI] 전문매핑 vs 결과통지방법코드 — 별개 설정 확인
- **상태**: 분석 완료, PL 보고 대기 (PL 반차)
- **배경**: PL 질문 — "계약대출 상품의 전문매핑이 1:1인데 N:1(1:N)이어도 문제없는지 확인"
- **핵심 발견: 전문매핑 ≠ 결과통지방법코드**
  - **전문매핑** (MCI 관리자페이지 기관관리): 요청 시 상품 묶음 방식, lnRqSqno 채번에 영향
    - 옵션: 1:1, N:1 (N:N 없음)
    - N:1 → lnRqSqno 동일 (1개 신청번호에 N개 상품)
    - 1:1 → lnRqSqno 상품별 다름
  - **결과통지방법코드** (우리 DB EFFC_TB_EF_FC_CM_AFLCN.RZT_NTC_METHC): inList2 상품목록 쿼리 방식 제어
    - 1=N:N → getApiDataTupwList1 (추가 파라미터 없음)
    - 2=1:1 → getApiDataTupwList2 (inqNo 추가)
    - 3=1:N → getApiDataTupwList3 (prdCode, wrsAmnC, wrsRqDtlDsc 추가)
  - **근거**: fco0009 — 전문매핑=N:1인데 RZT_NTC_METHC=2(1:1). 같은 거면 3이어야 하므로 별개
- **실데이터 비교 (전문매핑 차이)**:
  - fco0009 (N:1): 상품 5건(71221,71234,71203,71228,71239), lnRqSqno 전부 동일(2611200000168), tgrmNatvNo=6111200000141
  - fco1000 (1:1): 상품 2건(BL061,BL0692), lnRqSqno 각각 다름(2611200000179,2611200000181), tgrmNatvNo=6112000000149
  - fco1002 (1:1, 계약대출): 상품 1건(NHFiR6011), lnRqSqno=2611200000166 — 상품 1개라 차이 확인 불가
- **코드 경로 확인** (fcccb1100tS.java):
  - tgrCode="3" & fisCode="010" → 카드론
  - tgrCode="3" & fisCode="011" → 당행 (rztCode 분기 없음)
  - else → 대외계 (rztCode 분기 있음) ← fco0009, fco1000, fco1002 모두 여기
- **결론**: fco0009(N:1)에서 5건 정상 적재 확인됨 + fco1002와 같은 대외계 코드 경로 → 계약대출 N:1 변경해도 문제없음
- **증빙 방법**:
  1. fco1002 전문매핑 N:1로 변경 → 테스트 → 에러 없음 확인 (상품 1개라 N건 증명은 불가)
  2. fco0009 기존 데이터로 N건 정상 적재 증빙 (같은 코드 경로)
  3. 두 증빙 조합: "계약대출 N:1 에러 없음 + N건 적재 정상(같은 경로)"
- **한도조회관리내역(TB_EF_FC_CH_LMT_INQAMN) 적재 특징**:
  - N:1이든 1:1이든 최종 적재 모습은 동일 (상품별 각각 행)
  - 차이는 lnRqSqno가 같냐(N:1) 다르냐(1:1)
- **11/13 기록 수정**: 이전에 "LoReqtNo 채번 방식이 결과통지방법코드에 따라 다르다"고 기록했으나, 실데이터 확인 결과 lnRqSqno 채번은 전문매핑이 제어. 결과통지방법코드는 inList2 쿼리 방식만 제어
- **미확인**: 결과통지방법코드가 역거래(수신) 쪽에서도 분기하는지 (fctcb1200tS 등 확인 필요)
- **TODO**: PL 복귀 후 보고, fco1002 전문매핑 N:1 변경 테스트 실행

---

## 2026-11-13

### [MCI] 결과통지방법코드(RZT_NTC_METHC) 1/2/3 차이 분석 — PL 보고 완료
- **상태**: PL 보고 완료
- **배경**: PL 지시 — "1:N / N:N / 1:1 세 가지 차이가 뭔지 알아와라"
- **전체 구조**:
  - fisCodeList for문 → **2단계 분기**
  - 1차 분기: tgrCode(전문유형코드) + fisCode(금융기관코드)
    - tgrCode="3" & fisCode="010" → 카드론 (fctcb2100tS)
    - tgrCode="3" & fisCode="011" → 당행 (fctcb1100tS)
    - else → **대외계 MCI** ← 여기서만 rztCode 분기 진입
  - 2차 분기: rztCode(결과통지방법코드) — 대외계에서만 적용
    - 1 → N:N (ps315 = getApiDataTupwList)
    - 2 → 1:1 (ps316 = getApiDataTupwList2)
    - 3 → 1:N (ps317 = getApiDataTupwList3)
- **핵심 차이: LoReqtNo(한도조회일련번호) 채번 방식**:
  - N:N (rzt=1, ps315): 시퀀스 매번 새로 → 상품마다 독립 일련번호
  - 1:1 (rzt=2, ps316): #{inqNo} 고정값 → 전체가 같은 일련번호 공유
  - 1:N (rzt=3, ps317): FROM DUAL + Java 파라미터만 사용 (DB 테이블 안 탐)
- **ps310 CASE WHEN RZT_NTC_METHC='3'**: 분류가 아니라 데이터 준비. 1:N일 때만 fisCodeList에 상품코드(AFLCN_WRS_C) 미리 담아둠
- **검증서버 DB 조회 결과** (ps310 기준 19건):
  - rztCode=1 (N:N): 다수 — fco1003(440 NH생명), fco1002(449 NH손보) 등
  - rztCode=2 (1:1): 일부 — fco0006, fco0005, fco0001(011 당행) 등
  - rztCode=3 (1:N): **0건** — 검증서버에 해당 제휴사 없음
- **440(NH생명) 실제 경로**: tgrCode=3 → fisCode=440 → 대외계 else → rztCode=1(N:N) → ps315
- **당행(011) 참고**: rztCode=2이지만 1차 분기에서 당행 경로로 빠져서 rztCode 분기 안 탐
- **1:N 테스트 완료**: fco1002(449 NH손보) RZT_NTC_METHC=3으로 DB UPDATE 후 테스트
  - 송신(100번): 정상 (ps317 경로)
  - 접수확인(200번): 00000 정상
  - 결과통지(400번): 00000 Success — FCTCP1200I_OUT 정상 수신
  - DB 원복 완료 (RZT_NTC_METHC=1로 복원)
- **N:N 양쪽 제휴사 확인 완료**:
  - fco1003 (440 NH생명): 송신/접수/결과통지 전부 정상
  - fco1002 (449 NH손보): 송신/접수/결과통지 전부 정상
- **N:N/1:1/1:N 명칭 의미**: 앞=요청(가심사요청 IN) 건수, 뒤=결과통지(역거래 OUT) 건수
- **배운 점**:
  - fisCodeList for문은 제휴사 루프이고, rztCode 분기는 대외계(else) 안에서만 의미 있음
  - 당행(011)은 rztCode 값과 무관하게 1차 분기에서 전용 경로를 탐
  - ps310의 CASE WHEN은 분류가 아니라 1:N용 데이터 준비 (상품코드를 fisCodeList에 미리 셋팅)
  - DB에 rztCode 값이 있어도 Java 1차 분기에서 먼저 빠지면 해당 값은 dead data (당행 사례)
  - 1:N 테스트는 DB UPDATE(RZT_NTC_METHC=3)로 전체 체인(ps310→Java→ps317) 자동 전환 가능

### [BRIDGE] brpgUrl 올원앱 홈 이동 이슈 — 디버깅 결과 + PL 보고 완료
- **상태**: PL 보고 완료, 우리 팀 소관 아님 확인
- **디버깅 과정**:
  1. KFTC_ORG_C = "440" 확인 → gotoInsurance 정상 진입 (gotoNhWrc 아님)
  2. gotoInsurance 함수 내부 alert → brpgUrl 값 정상 수신
  3. svcId, prod, lnRqSqno, rq_gbn 전부 값 있음 (alert 확인)
- **발견**: 올원앱(appGbn="A") 경로에서 **brpgUrl을 사용하지 않음**
  - `callMoAppFunctionAfterClose('goService', {targetServiceId: 'ABFBP0001R', svcId, prod, lnRqSqno, rq_gbn: '02'})` 호출
  - brpgUrl은 else(스뱅) 경로에서만 `openWebPage(brpgUrl, ...)` 로 사용
- **gotoInsurance 함수 구조** (fcmap0106r.jsp):
  ```
  function gotoInsurance(brpgUrl) {
      var lnRqSqno = '${lnRqSqno}';
      if(appGbn == "A") {  // 올원앱
          KFTC_ORG_C 440 → svcId='01', prod='0100'
          KFTC_ORG_C 449 → svcId='02', prod='1000'
          callMoAppFunctionAfterClose('goService', {ABFBP0001R, svcId, prod, lnRqSqno, rq_gbn:'02'})
          // brpgUrl 안 씀
      } else {  // 스뱅
          openWebPage(brpgUrl, ...)  // brpgUrl 직접 사용
      }
  }
  ```
- **추가 발견 — ABFBP0001R = 금융그룹 홈 서비스**:
  - common.js line 1322: `/*NH농협카드 내 금융그룹 홈 버튼*/` → `$ABCOMMON.customNavigate('ABFBP0001R')`
  - 홈 버튼이 ABFBP0001R을 params 없이 호출 → 홈으로 이동
  - gotoInsurance도 같은 ABFBP0001R을 params 붙여서 호출 → params 유실/무시되면 홈처럼 동작
  - `afterCloseWebview` 검색 결과: 50건 전부 호출부, **정의부 없음** → 네이티브 SDK 주입 함수
- **JS→네이티브 체인 전체 추적 완료**:
  ```
  gotoInsurance(brpgUrl)
    → appGbn="A", KFTC_ORG_C="440" → svcId='01', prod='0100'
    → callMoAppFunctionAfterClose('goService', {ABFBP0001R, params})
    → NH_OFWebPopup.closeWebview() + callFunctionInWebview("MW", '$NH_OPA.moApp.afterCloseWebview', opt)
    → 🔒 네이티브 영역 (JS 추적 불가)
  ```
- **PL 보고 결과**:
  - PL: "세팅을 우리가 한 게 아니잖아" → ABFBP0001R 네이티브 서비스 설정은 우리 팀 소관 아님
  - PL: yml 파일(ti_config)에 스뱅 경로 URL만 설정하면 됨 (predev/dev/local은 TI 추가로 해결)
  - **결론**: 올원앱 홈 이동 이슈는 네이티브 쪽 문제, 우리 팀에서 더 할 건 없음
- **배운 점**:
  - 올원앱은 brpgUrl이 아니라 callMoAppFunctionAfterClose로 네이티브 서비스 호출 → brpgUrl에 파람 유무는 올원앱 경로와 무관
  - ABFBP0001R이 홈 서비스ID라는 것은 common.js의 `.home-link-move` 클릭 핸들러에서 확인 가능
  - `afterCloseWebview`는 네이티브 SDK가 웹뷰에 주입하는 함수 — JS 소스에 정의부 없으면 네이티브 영역
  - callMoAppFunctionAfterClose(closeWebview 후 호출) vs callMoAppFunction(바로 호출) — 다른 함수이므로 동작 차이 가능

### [MCI] 상품 테이블 구조 확인
- **상태**: 확인 완료
- **TB_EF_FC_CH_WRSAMN** 조회 결과:
  - fco0001(당행): PD001(올원직장인대출), PD002(NH직장인대출V), PD003(NH첫시작연대출) 등
  - fco1002(NH손보): HIRER01(보험계약대출) — 상품코드 체계가 당행(PD~)과 다름
  - fco1003(NH생명): 별도 코드 사용
- **배운 점**: 상품코드만으로 제휴사 구분 가능. 당행=PD~, NH손보=HIRER01 등 제휴사별 코드 체계 다름

### [BRIDGE] gotoFncorWrc 함수 — MT vs PT 코드 구조
- **상태**: 분석 완료
- **MT 파일**: fcmam0106r.jsp / **PT 파일**: fcmap0106r.jsp

#### gotoFncorWrc() 공통 흐름
1. e마케팅 사후처리 호출 (`callFireUserDefined2`)
2. `PRC_STSC` 체크 + `reqLoanproc` → 이미 처리됐으면 상단 분기로 바로 이동
3. 미처리 시 → `httpSender` (대출신청접수 전문 송신)
4. success 콜백 → 버튼 상태 변경 + brpgUrl로 페이지 이동

#### 상단 분기 (reqLoanproc=true, 재진입 시)
**MT**:
```javascript
if("${WRS_ENT_URLNM}" != "" || brpgUrl != ""){
    openWebPage("${WRS_ENT_URLNM}" == "" ? brpgUrl : "${WRS_ENT_URLNM}", function(ret){});
}
```
→ 제휴사 구분 없이 `openWebPage` 하나로 처리

**PT**:
```javascript
if("${map.KFTC_ORG_C}" == "011") {
    gotoNhrc("${WRS_ENT_URLNM}" == "" ? brpgUrl : "${WRS_ENT_URLNM}");
} else if("${map.KFTC_ORG_C}" == "440" || "${map.KFTC_ORG_C}" == "449") {
    gotoInsurance("${WRS_ENT_URLNM}" == "" ? brpgUrl : "${WRS_ENT_URLNM}");
} else {
    openWebPage("${WRS_ENT_URLNM}" == "" ? brpgUrl : "${WRS_ENT_URLNM}", function(ret){});
}
```
→ 제휴사별 네이티브 브릿지 분기 (011=gotoNhrc, 440/449=gotoInsurance, 기타=openWebPage)

#### httpSender success 콜백 — brpgUrl 이동 분기
**MT (svcId: FCLMM0102I, cusCtcChanC 하드코딩 "05")**:
```javascript
if(emptyUrlYn != "Y"){
    if("${map.KFTC_ORG_C}" == "011") {
        openWebPage(res.brpgUrl, function(ret){});        // 당행: 무조건 호출
    } else if(res.brpgUrl) {
        openWebPage(res.brpgUrl, function(ret){});        // 대외: brpgUrl 있을 때만
    }
}
```

**PT (svcId: FCLMP0102I, cusCtcChanC = appGbn분기 S→01/A→02)**:
- PT는 httpSender success 내에서도 011→gotoNhrc, 440/449→gotoInsurance, else→openWebPage 분기
- isNhOrg = ['010','011','440','449'].includes(kftcOrgC) 로 버튼 상태 분기

#### MT vs PT 핵심 차이
| 구분 | PT (앱) | MT (모바일웹) |
|---|---|---|
| 011 당행 | gotoNhrc() — 네이티브 앱 이동 | openWebPage() — 웹페이지 |
| 440/449 보험 | gotoInsurance() — 네이티브 서비스 | openWebPage() — 웹페이지 |
| 기타 대외 | openWebPage() | openWebPage() |
| cusCtcChanC | appGbn 분기 (S→01, A→02) | 하드코딩 "05" |
| svcId | FCLMP0102I | FCLMM0102I |

#### 011 vs else if 분기 이유 (MT)
- **011**: brpgUrl null 체크 없이 무조건 openWebPage — 당행은 MCI 안 타고 내부 세팅이라 항상 값 있음
- **else if**: `res.brpgUrl` truthy일 때만 — MCI 응답에서 brpgUrl 안 올 수 있음

### [LOCAL] MT 로컬 기동 시 404 에러 — 해결
- **상태**: 해결
- **증상**: 로컬 MT(8700) 홈 진입 시 콘솔에 `404 Not Found from POST http://127.0.0.1:8110/FCMAP0100R/getEvtListData`
- **원인**: MT(8700) → CM(8110)으로 `/FCMAP0100R/getEvtListData` 호출했으나 CM에 해당 엔드포인트 없음. CircuitBreaker FailedCalls: 6
- **해결**: CM 서버 관련 문제 — 사용자가 자체 해결

### [접근성] renderDivFormAccordionDom 코드 리뷰
- **상태**: 리뷰 완료, 수정 대기
- **파일**: 약관 동의 관련 JS (아코디언 DOM 렌더링 함수)
- **발견된 오류**:
  1. **(심각) 1depth 아코디언 `<button>` 태그 미완성**:
     - `>` 누락 — 태그 닫히지 않음
     - `onClick` 핸들러 없음 — 토글 불가능
     - `<span class="blind">펼치기</span>` content 없음
     - `</button>` 닫는 태그 없음
     - else 분기 버튼은 정상 (onClick + 펼치기 텍스트 + </button> 있음)
  2. **(경미) else 분기 `aria-describedby` trailing space**:
     - `' "'` → `"agChk_123 "` (끝에 불필요한 공백)
     - `' "'`를 `'"'`로 수정 필요

### 남은 할 일
- [x] **[MCI] 1:N 테스트**: fco1002 rztCode=3 세팅 → 송신/수신 정상 확인 → DB 원복 완료
- [x] **[MCI] 449(NH손보)** 결과통지 확인: 400번 00000 Success
- [x] **[BRIDGE] brpgUrl 디버깅**: 웹 쪽 정상 확인 + ABFBP0001R = 홈 서비스 발견
- [x] **[BRIDGE] PL 보고**: 네이티브 쪽 우리 소관 아님 확인, 스뱅 경로는 yml 설정으로 해결
- [x] **[MCI] PL 보고**: N:N/1:1/1:N 코드 분석 + 1:N 테스트 결과 보고 완료
- [ ] 면적 선택 팝업 (FCMIP2201P) JSP 코드에 tabindex + focus 적용
- [ ] ps111 쿼리 업종코드 CASE WHEN 수정 (fctcb01_sql.xml) — 4필드 방향 확정 후
- [ ] fcjim1400r.jsp PT: 차량번호 input aria-label 추가
- [ ] 퍼블에게 MT common.css 업데이트 요청 (flex-direction: row-reverse)
- [ ] ti_config.yml 운영환경(dr/run) URL 추가
- [ ] 배포리스트 엑셀 작성
- [ ] MT 실기기 테스트
- [ ] 스크래핑 연결 후 MCI 4필드 실데이터 재테스트
- [ ] 하드코딩 코드 전부 원복 (커밋 전)

---

## 2026-11-16

### [학습] 데이터 흐름 추적 1 — LN_RQ_JOB_C → jonCls
- **상태**: 완료
- **목적**: jonCls가 어디서 오는지 전체 흐름 추적 (폐쇄망 STS 실습)
- **흐름**:
  ```
  사용자 직업정보 입력 화면
    → fcjib0500sC → fcjib0500sS.updateJobInfo() → fcjib0500D
    → fcji_tb_ef_fc_cm_cmpr_rq_pu001_ea (UPDATE)
      UPDATE TB_EF_FC_CM_CMPR_RQAMN SET LN_RQ_JOB_C = #{ln_rq_job_c}
      (여기서 값 최초 세팅. INSERT 시점에는 비어있음)

    → fccob4000D.getLnRqJobC() SELECT로 읽어옴
    → fccob4000sS.getAmData()에서 tempMap.put("lnRqJobC", resultMap.get("LN_RQ_JOB_C"))
    → ps311 SQL: CMPR.LN_RQ_JOB_C AS "JonCls" (SQL alias로 변환)
    → TI에서 jonCls="5" 분기 → 사업자 MCI 전문 호출
  ```
- **값 의미**: "1"=직장인, "2"=공무원, "5"=사업자 (fccob4000sS switch문)
- **핵심 발견**: Java 변환이 아니라 SQL alias(`AS "JonCls"`)로 이름이 바뀜

### [학습] 데이터 흐름 추적 2 — SCRPG_SUC_YN
- **상태**: 완료
- **목적**: 스크래핑 성공여부가 어디서 세팅되고 어디서 읽히는지 추적
- **쓰는 곳 (UPDATE)**:
  - 파일: fccob02_sql.xml
  - SQL ID: `fccob_tb_ef_fc_cm_cmpr_rq_pu001_ea`
  - `UPDATE TB_EF_FC_CM_CMPR_RQAMN SET SCRPG_SUC_YN = #{SCRPG_SUC_YN}, PRG_STP_DSC = '06'`
  - 이 UPDATE는 **범용 쿼리** — `<if test>` 동적SQL로 호출자마다 다른 컬럼 세팅
  - 호출자: 건강보험스크래핑(fccrbNHISScrapingService), 홈택스스크래핑, 직업/자산정보 저장 등
- **읽는 곳 (SELECT)**:
  - ps311 SQL: `NVL(CMPR.SCRPG_SUC_YN, 'N') as "ScrpInfoYN"`
  - TI 가심사요청 전송 시 스크래핑 완료 여부 판단용
- **배운 점**: 하나의 UPDATE를 여러 서비스가 공유하는 구조. 파라미터에 뭘 넣느냐로 동작이 달라짐

### [접근성] 퍼블 요청 3건 — 작업 진행중
- **상태**: 진행중

#### 1. 약관 상세보기 title 수정
- **파일**: fcagm0100r.jsp (line 322~323)
- **변경**: `setChecked` 함수 안 `.attr('title', ...)` 삼항연산자 → `'약관 상세보기'` 하나로 통일
- **기존**: `isOn ? '상세 동의 팝업 열기 선택됨' : '상세 동의 팝업 열기 선택안됨'`
- **변경후**: `'약관 상세보기'`
- **상태**: 수정 완료, 테스트 필요

#### 2. 페이지 타이틀 — 페이지별 상세명 + NH금융상품비교
- **공통 템플릿**: template-main.jsp (line 28)
  - `<title><c:if test="${not empty RESULT_MAP.pageTitle}">${RESULT_MAP.pageTitle} </c:if>NH 금융상품비교</title>`
- **Tiles 설정**: nh-mobile-fc-definition.xml — 와일드카드 매핑(`/fc/*/*.jsp`)이라 페이지별 definition 없음
- **방법 1 (전용 컨트롤러 있는 페이지)**: `resultMap.getOut().put("pageTitle", "페이지명")`
  - 예: fcmam1100sC → 적용 성공 확인됨
  - fclmm0100sC (비교조회결과) → resultMap 받은 직후 한 줄 추가
- **방법 2 (defaultPageViewController / denyMap 컨트롤러)**: JSP에서 `document.title` JS로 세팅
  - denyMap 리턴하는 컨트롤러는 RESULT_MAP 전달 안 됨 (fcwsm0700sC, fccom2001sC 확인)
  - defaultPageViewController인 페이지도 전용 컨트롤러 없어서 JS 방식
  - `<script>document.title = "페이지명 NH금융상품비교";</script>` — `<c:if>` 바깥에 넣을 것
- **주의**: template-main.jsp에서 이미 "NH금융상품비교" 붙이므로, 컨트롤러 방식은 pageTitle에 "NH금융상품비교" 안 넣기. JS 방식은 전체 title을 직접 세팅하므로 붙여야 함
- **상태**: 일부 페이지 적용 완료, 나머지 페이지 + 테스트 필요

#### 3. 레이아웃 팝업 — slide 대신 modal-window
- **상태**: 미착수
- **내용**: 퍼블이 slide 팝업을 modal-window로 변경 요청

### [내일 체크리스트] 11/17
- [ ] 약관 상세보기 title 수정 — 로컬/dev 테스트
- [ ] 페이지 타이틀 — 적용한 페이지들 F12에서 `<title>` 확인
- [ ] 페이지 타이틀 — 나머지 대상 페이지 작업 (퍼블한테 전체 목록 확인)
- [ ] 레이아웃 팝업 slide→modal-window — 퍼블한테 대상 팝업 확인 후 작업
- [ ] 퍼블한테 각 페이지 정확한 타이틀명 확인받기

---

## 2026-11-17

### [카카오 알림톡] MT용 카카오 알림톡 추가 — 코드 분기 작성중
- **상태**: 진행중
- **배경**: 동료(PC 고장)한테 넘겨받은 작업. MT 전용 카카오 알림톡 추가
- **완료 사항**:
  - 관리자페이지 상세코드 4건 등록 (01=스뱅, 02=올뱅, 03=비교금융, 04=모바일웹)
  - 쿼리(ps003) UNION ALL 4개 ORD 추가
  - 타이틀/본문 세팅 완료
- **코드 분기 로직** (fccob1030sS.java):
  - cusCtcChanC = dataMap.getIn().getValue("cusCtcChanC", "") — 고객접촉채널코드
  - 비회원("12") 분기:
    - MT("05") → 4번 ORD만 발송
    - PT(그 외) → 1번 ORD만 발송
  - 기존고객 → 2번, 3번 ORD만 발송
  - **주의**: DB에 "05"로 저장됨 (zero-padding). `"5".equals("05")`는 false → 반드시 `"05"` 사용
- **남은 것**: 코드 적용 후 dev 배포 + 테스트

### [퍼블 요청] 약관 팝업 after→body 변경 + 접근성 포커스
- **상태**: 완료
- **파일**: fcagm0103p.jsp (약관동의 팝업, MT)
- **변경 내용**:
  1. `$('#signTermsArea').after(html)` → `$('#termsModalBody').html(html)` (id 기반 셀렉터)
  2. `<div class="modal-body">` → `<div class="modal-body" id="termsModalBody" tabindex="-1">` (id+tabindex 추가)
  3. success 콜백에 `common_closeInitLodingBar()` + `setTimeout(300ms)` + `$('#termsModalBody').focus()` 추가
- **이유**: `.modal-body` 클래스가 페이지에 2개 존재 → class 셀렉터 대신 id 셀렉터 필요
- **after vs html 차이**: `.after()` = 형제로 바깥에 삽입, `.html()` = 자식으로 내부 교체

### [퍼블 요청] 페이지별 타이틀 추가 — 일부 적용
- **상태**: 진행중
- **방법 1 (컨트롤러 있는 페이지)**: `resultMap.getOut().put("pageTitle", "페이지명")` → template-main.jsp `<title>` 반영
- **방법 2 (denyMap/기본 컨트롤러 페이지)**: JSP에서 `<script>document.title = "페이지명 NH금융상품비교";</script>`
- **적용한 페이지**: fcmam1100sC 등 일부 완료
- **남은 것**: 나머지 페이지 + 퍼블한테 전체 목록/정확한 타이틀명 확인

### [퍼블 요청] 레이아웃 팝업 slide→modal-window — 셀렉터 수정 필요
- **상태**: 진행중
- **내용**: 퍼블이 slide 팝업을 modal-window로 변경 → 기존 `.slide .modal-window` JS 셀렉터가 null 리턴
- **수정 패턴**: `.slide .modal-window` → `#팝업ID .modal-window`
- **적용 완료**: 공유하기 팝업 — `#FCCOP0120P .modal-window` (fcmap0100r.jsp)
- **남은 것**: 다른 팝업도 동일 패턴 점검 필요

### [접근성] 공유하기 팝업 포커스 이동 — 셀렉터 수정
- **상태**: 완료
- **파일**: fcmap0100r.jsp (share 버튼 클릭 함수)
- **변경**: `.slide .modal-window` → `#FCCOP0120P .modal-window`
- **패턴**:
  ```javascript
  fcPop.modalOpen('FCCOP0120P', {}, function() {
      window.setTimeout(function(){
          var el = document.querySelector('#FCCOP0120P .modal-window');
          if(el) { el.focus(); }
      }, 100);
  }, this);
  ```
- **확인**: `querySelectorAll('.modal-window').length` = 2개 → id 기반 셀렉터 필수

### [Git] Merge conflict 해결 2건
- **상태**: 완료
1. **fccrm0400sC.java (휴대폰 본인인증 컨트롤러)**:
   - HEAD: 수동 `SpringContext.getProperty(...)` + pageTitle
   - Incoming: `EnvUtil.isCbt()` (더 깔끔)
   - **해결**: pageTitle 유지(내 작업) + EnvUtil.isCbt() 채택 + SpringContext 라인 제거
2. **fcagm0103p.jsp (약관동의 팝업)**:
   - HEAD: `$('.modal-body').html(html)` (내 변경)
   - Incoming: `common_closeInitLodingBar()` + `$('#signTermsArea').after(html)`
   - **해결**: `$('#termsModalBody').html(html)` 유지 + `common_closeInitLodingBar()` 추가 (로딩바 짝 맞추기)
- **SourceTree 주의**: "내것으로 변경" = HEAD만 유지 (상대 코드 전부 삭제). 충돌 부분 수동 편집 후 "충돌 해결됨으로 표시" → 스테이지 → 커밋

### [내일 체크리스트] 11/18
- [ ] **[카카오 알림톡]** cusCtcChanC 분기 코드 적용 → dev 배포 → 테스트
  - "05" zero-padding 확인, 비회원/기존고객 분기 동작 검증
- [ ] **[퍼블 요청]** 페이지 타이틀 — 나머지 대상 페이지 작업
  - 퍼블한테 전체 목록 + 정확한 타이틀명 확인받기
  - 적용한 페이지 F12에서 `<title>` 확인
- [ ] **[퍼블 요청]** 레이아웃 팝업 slide→modal-window
  - 퍼블한테 대상 팝업 목록 확인
  - `.slide .modal-window` 셀렉터 사용하는 JS 전체 검색 → `#팝업ID .modal-window`로 수정
- [ ] **[퍼블 추가]** 스크래핑 중 문구 수정 + 휴대폰 인증 화면 타이틀 수정 (급하지 않음)
- [ ] **[접근성]** 전체 팝업 포커스 이동 검토
  - fcPop.modalOpen 공통 함수에 focus 처리 여부 확인
  - 없으면 개별 팝업마다 setTimeout+focus 패턴 적용
- [ ] **[기존]** 면적 선택 팝업 (FCMIP2201P) tabindex + focus 적용
- [ ] **[기존]** 배포리스트 엑셀 작성
- [ ] **[기존]** MT 실기기 테스트
- [ ] **[기존]** 하드코딩 코드 전부 원복 (커밋 전)

### [취약점] 취약점 결과보고서 개선 — 11/18 할당

**점검 대상**: smartdev.nonghyup.com:8700, 28700

#### WEB-SER-019 (불충분한 이용자 인증) / FCCRM0614P
- **현상**: A계정으로 휴대폰 본인인증 후 카카오 인증서 클릭 → 응답 값에서 정보를 B로 변조(Burp Suite 프록시로 HTTP 응답 가로채기) → 인증 요청 메시지를 받을 휴대폰 정보를 B계정으로 수정 → B의 핸드폰으로 지문인증 시 공마데 조회됨 → 조회결과가 A계정으로 나옴
- **변조 방식**: Burp Suite로 서버→브라우저 응답 가로챔 → birthDayFull, authMpno, cusno 값을 B의 정보로 변경 → 변조된 응답이 브라우저에 전달
- **근본 원인**: 클라이언트에서 올라온 본인정보(cusno, birthDayFull, authMpno)를 서버가 그대로 신뢰. 세션에 저장된 본인 정보와 대조 검증 없음
- **개선 방향**: 서버 세션에 저장된 인증 정보와 요청 파라미터의 본인 정보를 서버 사이드에서 대조 검증. 클라이언트 파라미터 대신 세션 값으로 대체하거나, 불일치 시 거부
- **조사 필요**: FCCRM0614P에서 cusno/birthDayFull/authMpno가 파라미터로 오는지 세션에서 꺼내는지 추적
- **상태**: 연구중 (방식 보고 후 개발)

#### WEB-SER-021 (무제한 요청 차단) / FCCRM0610R
- **현상**: 카카오 인증 선택 후 휴대폰 정보 입력 및 약관동의 진행 → 카카오 인증 요청 값을 Intruder로 이동 및 30회 반복 요청 세팅(1초 1회) → 반복요청시도 가능 → 카카오 인증요청이 30회 전송됨
- **근본 원인**: 카카오 인증 요청 API에 횟수/시간 제한 로직 없음
- **개선 방향**: 세션 기반 카운터(세션에 요청 횟수 + 최초 요청 시간 저장 → N분 내 M회 초과 시 차단) 또는 DB 기반 이력 저장 후 차단
- **담당**: 다른 팀원이 개발 중
- **상태**: 코드 추적 완료 (흐름 파악)

##### 021 코드 추적 결과

**전체 흐름**:
```
[JSP] fccrm0614p.jsp — 카카오 등 인증 버튼 클릭
  → oneClickBind() 실행
  → fcPop.modalOpen('FCAGM0102P') — 약관동의 팝업
  → goAuth() — 약관 체크 확인 후 팝업 닫기 (인증 요청 아님)
  → 콜백에서 param 세팅 (cusno, cmprRqSqno 등 EL 표현식)
  ↓
[JS] httpSender({ svcId: 'FCJIM2500I', param }) — 서버 첫 요청
  ↓ 성공 시
[JS] simpleAuthLoginGbnServerScrap(atoaTpcNm) — FCCRM0610R 호출
  ↓
[Controller] fccrm0605sC.doSimpleCertScrapping(FrontData)
  → inq_dsc = "1" 분기 → fccrm0605sS.getLoginRequest(dataMap)
  ↓
[Service] fccrb0604sS.getLoginRequest() — 건보공단 간편인증 로그인 실행
  → PETRA 암호화 후 외부 API 전송
```

**rate limit 적용 위치**: `doSimpleCertScrapping` 메서드 내 `inq_dsc="1"` 분기, `getLoginRequest` 호출 전
- 프론트의 `isRequesting` 플래그는 Burp Suite Intruder로 우회 가능 → 서버 쪽에서 차단 필수
- 로그아웃 등 다른 분기는 제한 불필요 → `inq_dsc="1"` 안에만 적용

**세션 API (프로프레임웍)**:
- 저장: `SessionManagerFC.putTempValue(String key, Object value)` — LoginSession 내 임시값 저장
- 읽기: `SessionManagerFC.getTempValue(String key)` → Object 반환
- 내부 구조: `HttpSession` → `LoginSession` 객체 → `putTempValue`/`getTempValue`
- 관련 클래스: `SessionManagerFC.java` (com.nhbank.ef.fc.base.pt.session), `SessionHelper` (HttpSession 직접 접근)

#### NH-WEB-052 (중요정보 마스킹 처리 여부) / FCCRM0600R, FCCRM0614P
- **현상**: 인증서 선택 화면 진입 시 응답 값에 실명정보 콘솔로 노출 → 인증서 선택 후 다음 단계 진입시에도 같은 노출
- **근본 원인**: 서버 응답에 주민번호 평문이 포함되어 브라우저 콘솔에서 확인 가능
- **개선 방향**: 서버에서 클라이언트로 보내기 전에 마스킹 처리 (예: 850115-1****** 뒤 6자리 별표). 전체 값이 필요한 로직이면 세션에서 처리하고 화면에는 마스킹된 값만
- **마스킹 유틸 발견**: `NHBFormatUtil.maskWithPattern(String targetStr, String pattern)` — 프로프레임웍 공통 유틸 (`nhbank/ef/fc/commons/util/NHBFormatUtil.java`)
  - `?` = 원본 글자 그대로, `*` = 별표 마스킹, 그 외 문자 = 그대로 출력
  - 주민번호 예: `maskWithPattern("8501151234567", "??????-?******")` → `850115-1******`

##### 052 코드 추적 결과

**전체 흐름**:
```
[세션] custInfo.getRLNO() (평문 주민번호)
  ↓
[Controller] fccrm0600sC.java line 76: rlno = custInfo.getRLNO()
  ↓ (프레임워크 자동 바인딩 추정)
[JSP] fccerCertCommonScrap.jsp line 87: params.RLNO = "${rlno}"  ← 평문 노출 지점
  ↓
[JS] httpSender로 FCCRM0611R 서비스 호출 (params에 평문 포함)
  ↓
[Service] fccrb0604sS.java line 280: PetraCipherUtil.encrypt(RLNO) → 암호화
  ↓
[외부] 건보/국세청 API로 암호화된 값 전송
```

**취약점 지점**: JSP에서 `${rlno}`로 평문이 브라우저에 도달 → Network/Console 탭에서 확인 가능
**서버는 이미 암호화해서 사용 중**: fccrb0604sS에서 PETRA 암호화 후 외부 전송
**문제**: 서버가 세션에 주민번호를 갖고 있으면서도 클라이언트를 거쳐서 받는 구조

**조치 방향**:
  - JSP에 `${rlno}` 평문을 내려보내지 않고, 서버(fccrb0604sS)에서 세션으로 직접 가져와서 처리
  - 또는 JSP에 내려줘야 한다면 NHBFormatUtil.maskWithPattern으로 마스킹 후 전달
  - 단, 마스킹하면 건보/국세청 인증에 실제 주민번호가 필요하므로 → 서버 직접 처리 방식이 근본 해결
- **상태**: 조치 완료 (11/19)
- **조치 내용**: PL이 JSP에서 loginByFinCert 함수 및 params.RLNO 삭제, fccom2006R/2007R도 삭제
- **확인 결과**: Network 탭에서 평문 미노출 확인, 인증 정상 동작
- **ServerScrap JSP**: 동일 함수 존재하나 테스트서버 조건문(isLocal/isPredev/isDev) 내에서만 RLNO 세팅 → 운영 미노출
- **PT(fccop2006.jsp)**: 동일하게 테스트 조건문 내에서만 세팅 → 운영 미노출
- **Service(fccrb0604sS.java)**: 변경 없음 — 삭제된 함수(loginByFinCert)의 호출 경로 없음 확인

**연구 순서**: 052(완료) → 021(흐름 파악 완료, 다른 팀원 개발 중) → 019(PL 조치 완료)

---

## 2026-11-19

### [접근성] 페이지 타이틀 추가
- **상태**: 완료
- **작업 내용**: 퍼블 요청에 따라 각 화면에 document.title 추가
- **적용 파일 및 타이틀**:
  - `fccom2001i.jsp` → "공공마이데이터로딩 | NH금융상품비교" (고정)
  - `fccom2005i.jsp` → lnRqJobC 삼항연산자 (국세청로딩 / 공공마이데이터로딩)
  - `fccom2006i.jsp` → lnRqJobC 삼항연산자 (국세청로딩 / 공공마이데이터로딩)
  - `fcmam0100r.jsp` setBlockMsg → "점검중 | NH금융상품비교"
  - `header.jsp` modalOpen 214,288줄 → "설문조사 | NH금융상품비교" (열기 직전 변경, 콜백 원복)
  - `fcPopup.js` appError → "로그인연장오류 | NH금융상품비교" (연장 실패)
  - `fcPopup.js` modalSessionTimeOutOpen → "로그인연장오류 | NH금융상품비교" (세션 타임아웃)
  - `fcmlm2400r.jsp` goSubMain → "금융결전산오류안내 | NH금융상품비교"
  - `fcmam0100p.jsp` → "이벤트상세 | NH금융상품비교" (ready에서 변경, 닫기 콜백 원복)
- **타이틀 적용 규칙 (통일)**:
  1. 일반 페이지 → `document.title = "...";`
  2. 레이어팝업(모달) → 열기 직전 변경, 닫힐 때 originalTitle로 원복
  3. `modalAlertOpen` 첫 번째 파라미터에는 넣지 않음 (팝업 헤더용이라 용도 다름)

### [접근성] aria-describedby → aria-labelledby 전체 치환
- **상태**: 완료
- **작업**: VS Code Ctrl+Shift+H로 JSP 파일 대상 전체 치환
- **결과**: 44건 / 15파일 치환 완료
- **제외**: *.min.js(라이브러리), *.html(퍼블 원본)
- **검증**: *.js 검색 시 커스텀 JS에 aria-describedby 미사용 확인 → 영향 없음
- **로컬 테스트**: 약관동의 등 체크박스 화면 정상 동작 확인

### [취약점] WEB-SER-019 — PL 조치 완료 확인
- **상태**: PL 조치 완료
- **파일**: fccrm0605sC.java
- **조치 내용**:
  1. `checkCustInfo` 메서드 추가 — 프론트 파라미터(이름/생년월일/전화번호/통신사)를 세션 고객정보와 비교, 불일치 시 요청 거부
  2. RLNO: `EnvUtil.isProd()||isDr()` 조건 제거 → 모든 환경에서 세션 값 사용 (052도 겸함)
  3. 적용 위치: inq_dsc="1" 로그인요청 + processSimpleAuth
- **검증 로직**: 주민번호 7번째 자리로 세기 판별(1,2,5,6→19XX / 3,4,7,8→20XX / 9,0→18XX) → 생년월일 완성 후 비교
- **lnRqJobC != "5"** (국세청 아닌 경우)에만 이름/생년월일/전화번호/통신사 검증

---

### [퍼블] 주민등록번호 input→div 구조 변경 — 보류
- **파일**: fccm0400r.jsp
- **요청**: 주민번호 뒤 7자리 input을 div(role="button" + data-attr) 구조로 변경
- **보류 사유**: transkey(보안키보드)가 input 태그에 바인딩 → div로 바꾸면 `allocate` 에러 발생. hidden input 유지 필요 여부 등 퍼블/PL 협의 필요

---

## 2026-11-20 (오후)

### [접근성] fcPopup.js 팝업 포커스 관리 — MT/PT 공통

- **상태**: 완료 (modalOpen 포함 전체 해결)
- **핵심 원인**: `focus()`를 jQuery `fadeIn()` 애니메이션 도중에 호출하면 포커스가 안 붙음. fadeIn 콜백(애니메이션 완료 후)에서 호출해야 함
- **수정 파일**: MT/PT 각각 `fcPopup.js` (`MT/src/main/webapp/static/service/js/common/fc/fcPopup.js`)

#### 함수별 수정 내용

| 함수 | 방식 | 포커스 대상 | 상태 |
|------|------|------------|------|
| modalAlertOpen | fadeIn callback | `.button-area button:first` | ✅ MT/PT |
| modalConfirmOpen | fadeIn callback | `.button-area button:first` | ✅ MT/PT |
| modalConfirmOpen2 | fadeIn callback | `.button-area button:first` | ✅ MT/PT |
| alertOpen | fadeIn callback | `.button-area button:first` | ✅ MT/PT |
| modalOpen | 기존 setTimeout(500) 안에서 focus | `.modal-window:first` (tabindex="-1") | ✅ MT/PT |

#### modalOpen이 다른 이유
- modalOpen은 AJAX로 콘텐츠 로드 후 `addClass("slide")` CSS 애니메이션 + `.show()` 사용
- `.show()` 후 `.fadeIn()`은 이미 보이는 요소라 **no-op** → 콜백 안 불림
- 해결: fadeIn 콜백 대신 기존 `window.setTimeout(function(){...}, 500)` 블록 안에서 focus 처리

#### 수정 패턴 (fadeIn 계열)
```javascript
// BEFORE:
$('#element').fadeIn();
$('#element').focus();  // ← 애니메이션 중이라 안 먹힘

// AFTER:
$('#element').fadeIn(function(){
    $(this).find('.button-area button').first().focus();
});
```

#### 수정 패턴 (modalOpen — setTimeout 계열)
```javascript
// 기존 setTimeout(500) 안, attr 설정 후:
$('#' + serviceId).find('.modal-window').first().attr('tabindex', '-1').focus();
// fadeIn 콜백 블록은 삭제
```

#### PT 문법 에러 사고
- **증상**: PT 전체 화면 안 열림, `ReferenceError: httpSender is not defined`
- **원인**: `modalConfirmOpen2`에서 `function()` 누락 → JS 문법 에러 → fcPopup.js 전체 로드 실패 → 모든 함수 사용 불가
```javascript
// 에러 코드:
$('#fcCommConfirmPop2').fadeIn(
    $(this).find('.button-area button').first().focus();  // function(){ 빠짐
});
// 올바른 코드:
$('#fcCommConfirmPop2').fadeIn(function(){
    $(this).find('.button-area button').first().focus();
});
```
- **교훈**: 공통 JS 파일 수정 시 문법 에러 하나가 전체 사이트를 먹통으로 만듦. fadeIn 콜백은 반드시 `function(){` 포함 확인

### [접근성] fccrm0400r.jsp 체크박스 title 접근성 수정

- **상태**: 완료
- **파일**: `MT/src/main/webapp/WEB-INF/jsp/service/fc/cr/fccrm0400r.jsp`
- **문제**: 체크박스 해제 후 재체크 시 span title이 "상세 동의 선택안됨"에서 "선택됨"으로 안 돌아옴
- **수정 3곳**:
  1. **개별 체크박스 click handler**: `else { $chk.closest(".form-check").find("span").attr("title", "상세 동의 선택됨"); }` 추가 + `chkAll()`을 if/else 밖으로 이동
  2. **전체동의(#agCheckAll) change handler**: else 분기 추가 — `$chk.prop("checked", true)` + `.attr("title", "상세 동의 선택됨")`
  3. **팝업 콜백 (약 308줄)**: `$chk.closest(".form-check").find("span")` 뒤에 `.attr("title", "상세 동의 선택됨")` 추가 (불완전한 체인 수정)
  4. **전체 체크 영역**: 전체동의 체크/해제 시 title 반영 추가

### 남은 할 일
- [ ] PT fcPopup.js modalOpen 동일 패턴 적용 확인
- [ ] 전체 팝업 MT/PT 테스트 (modalAlert, modalConfirm, modalConfirm2, alertOpen, modalOpen)
- [ ] 접근성 수정 커밋 전 임시코드/console.log 전부 삭제
- [ ] fco1002 전문매핑 N:1 변경 테스트 (PL 복귀 후)
- [ ] 배포리스트 엑셀 작성
- [ ] MT 실기기 테스트
- [ ] 하드코딩 코드 전부 원복 (커밋 전)

---

## 2026-11-23

### [보안] 취약성검사 — 본인인증 세션 불일치 취약점 (fccrm0605sC.java)
- **상태**: PL 수정 완료, 코드 분석 완료
- **취약점**: A 사용자 로그인 → B 사용자로 본인인증 → A의 대출조회 데이터 노출
- **원인**: 프론트에서 올라온 RLNO/이름/생년월일을 서버 세션값과 비교하지 않고 그대로 사용
- **PL 수정 (3가지)**:
  1. **RLNO 강제 덮어쓰기** (setCustInfoToDataMap line 179, doHomeTaxScrapping line 231):
     - 기존: `if(EnvUtil.isProd() || EnvUtil.isDr() || "".equals(RLNO))` 조건부 세팅
     - 수정: 조건 삭제, **무조건** `dataMap.getIn().setValue("RLNO", custInfo.getRLNO())` — 프론트 값 무시하고 세션값 덮어씀
  2. **checkCustInfo 게이트 추가** (line 67, 313):
     - `if(!checkCustInfo(dataMap))` → 에러코드 "NCC0C00001" 리턴 + 로직 차단
     - 로그인요청(inq_dsc="1")과 간편인증(processSimpleAuth) 양쪽에 적용
  3. **checkCustInfo 메서드 신규 (line 373~)**:
     - 세션: `SessionManagerFC.getCustInfo()` → 이름(CUS_NM), 주민번호(RLNO→생년월일), 전화번호(MpsvNo+MphNo+MpsqNo)
     - 프론트: `dataMap.getIn().getValue("username"/"userbirthday"/"userphone"/"usertelcd")`
     - 4개 값 비교: 하나라도 불일치 → `return false` → 차단
     - 사업자(lnRqJobC="5")는 체크 스킵 (별도 인증 체계)

#### 대안/보완 방법 검토 (PL 지시)

**PL 수정 방식의 약점:**
1. **클라이언트 비교값 조작 가능**: checkCustInfo가 비교하는 username/userbirthday/userphone은 프론트에서 올라오는 값. 공격자가 A로 로그인한 뒤 프론트 값도 A의 정보로 맞추면 체크를 통과할 수 있음 → 단, RLNO 덮어쓰기(수정1)가 있으니 RLNO 기반 공격은 막힘
2. **사업자(jonCls="5") 스킵**: 사업자 경로에서는 checkCustInfo 체크 없음. 사업자로 위장 가능한지 확인 필요
3. **checkCustInfo 실패 시 에러 메시지**: "NCC0C00001"만 리턴 — 사용자에게 뭐가 틀렸는지 안 알려줌 (보안상 이게 맞음)

**다른 방식들:**
1. **토큰 기반 검증**: 본인인증 완료 시 서버에서 1회용 토큰 발급 → 이후 요청에 토큰 포함 → 세션의 cusNo와 토큰의 cusNo 일치 체크. 프론트 값 비교보다 안전하지만 구현 복잡도 높음
2. **본인인증 결과를 세션에 바인딩**: 본인인증 성공 시 인증된 CI/DI 값을 세션에 저장 → 이후 조회 시 세션의 CI와 로그인 사용자의 CI가 같은지 체크. 가장 근본적이지만 CI 연동 필요
3. **인증 완료 후 세션 재생성**: 본인인증 성공 시 기존 세션 무효화 + 인증된 사용자 정보로 새 세션 생성. 세션 불일치 자체가 불가능해짐. 단, 기존 세션 데이터(장바구니 등) 유실 위험
4. **요청 시점 DB 비교**: 프론트 값이 아니라 DB에서 cusNo 기준으로 이름/생년월일/전화번호를 직접 조회해서 비교. 프론트 조작 불가. 단, DB 조회 추가 비용

- **현재 PL 방식 평가**: 실용적이고 빠른 수정. RLNO 덮어쓰기 + 프론트값 비교 조합으로 대부분의 시나리오 차단. 완벽하진 않지만 비용 대비 효과적
- **가장 근본적인 해결**: 2번(CI/DI 세션 바인딩) 또는 4번(DB 직접 비교). 프론트 값에 의존하지 않으므로 조작 불가

### 남은 할 일
- [ ] PT fcPopup.js modalOpen 동일 패턴 적용 확인
- [ ] fco1002 전문매핑 N:1 변경 테스트 (PL 복귀 후)
- [ ] 배포리스트 엑셀 작성
- [ ] MT 실기기 테스트
- [ ] 하드코딩 코드 전부 원복 (커밋 전)
- [ ] 접근성 수정 커밋 전 임시코드/console.log 전부 삭제
- [ ] CBT 정상화 후 사업자 MCI 테스트 (금요일 예정)
- [ ] MT 011 if문 제거 적용 (PL 확인 후)

---

## 2026-11-24

### [BRIDGE] MT 당행상품 about:blank — 해결
- **상태**: 해결
- **문제**: MT에서 당행(011) 상품 클릭 시 about:blank 빈 화면
- **원인 분석**:
  - `TIUtil.getNhbUrl(String chnlCd, String itemCd)` 메서드 확인 (line 1025):
    ```java
    String tempKey = "J011.URL." + safeTrim(chnlCd) + "." + safeTrim(itemCd);
    return StringUtils.trimToEmpty(SpringContext.getProperty(tempKey));
    ```
  - 키 `J011.URL.03.11` 조합, 값 없으면 빈문자열("") 리턴
  - common_ti_config.yml dr 환경 URL을 `https://nhlink.nonghyup.com`으로 수정
- **해결**: URL 수정 → 배포 → CBT(MT)에서 정상 페이지 이동 확인
- **getNhbUrl 호출 패턴 비교**:
  | 파일 | 호출 |
  |---|---|
  | fctcb1300tS (fc) | `getNhbUrl("03", "11")` — 하드코딩 |
  | fcdcb1300tS (dc) | `getNhbUrl(cusCtcChanC, aflcnWrsC)` — 동적 |
  | fctcb1400tS | `getNhbUrl("011", aflcnWrsC)` — 혼합 |

### [BRIDGE] MT 011 if문 제거 — 완료
- **상태**: 수정 완료 + alert 테스트 PASS + 기획서 확인
- **파일**: fcmam0106r.jsp
- **변경 전**:
  ```javascript
  if("${map.KFTC_ORG_C}" == "011") {
      openWebPage(res.brpgUrl, function(ret){});      // 당행: brpgUrl 체크 없음
  }else if(res.brpgUrl) {
      openWebPage(res.brpgUrl, function(ret){});      // 제휴사: brpgUrl 체크 있음
  }
  ```
- **변경 후**:
  ```javascript
  if(res.brpgUrl) {
      openWebPage(res.brpgUrl, function(ret){});
  }
  ```
- **PL 지시**: if문 지우고 else if만 살려도 문제없는지 확인
- **검증**:
  1. 코드 분석 — 두 분기 동작 동일(openWebPage). 011 분기 제거 시 당행에도 brpgUrl null 체크 추가됨 → about:blank 방지
  2. alert 테스트 — `alert("brpgUrl: " + res.brpgUrl)` 값 정상 표시, URL로 페이지 이동 정상 동작
  3. **기획서 확인** — "당행(NH은행): 모바일브랜치로 이동 / 제휴사: 해당 제휴사로 이동". 서버단(fctcb1300tS)에서 당행은 `getNhbUrl("03","11")`=모바일브랜치 URL, 제휴사는 제휴사 URL로 brpgUrl 세팅 → 프론트엔드에서는 brpgUrl 하나로 openWebPage 호출이 기획 의도와 일치
- **결론**: 기획서 의도 충족 + 코드 안전성 향상 (null 체크 통일)

### [MCI] 사업자 MCI 로컬 테스트 — 송수신 성공
- **상태**: 로컬 성공, 검증서버 대기
- **스크래핑 하드코딩 (fccrb0604sS.java)**:
  - `PetraCipherUtil.encrypt()` 로컬에서 예외 발생 → line 83도 주석 필요
  - 로컬은 PETRA 미동작 → 암호화된 주민번호 직접 하드코딩
  - `inputMap.put("주민등록번호", "암호화값")` — line 85
- **MCI 바디 하드코딩 (fctcb1101tS.java)**:
  - `cusnm` ≠ 주민번호. `pwize_tr_rlno`가 암호화거래실명번호 (line 300)
  - `mciDataMap.getIn().setValue("pwize_tr_rlno", "암호화값")` — 여기에 하드코딩
  - 주의: `setMciBody`와 `setApiBody` 두 곳에서 세팅 — 한쪽만 바꾸면 덮어써짐
- **로컬 결과**: MCI 송신 + 역거래 응답 수신 확인
- **검증서버(CBT)**: 현재 CBT 환경 막힘 (한도 미수신 상태). 금요일 정상화 예정
- **CBT 아빠계정 테스트**: MCI 아예 안 나감 + 송수신이력 테이블에도 없음 → CBT 환경 차단이 원인
- **커밋 전 원복 필수**: 주민번호 하드코딩(fccrb0604sS, fctcb1101tS), PetraCipherUtil 주석, 디버그 로그 전부

### [CONFIG] 전자인증서 pageTitle 추가 — 방법 확정
- **상태**: 방법 확정, 적용 대기
- **파일**: fccrm0600sC.java (컨트롤러), fccrm0614p.jsp (JSP)
- **현재**: 컨트롤러에 `pageTitle = "전자서명확인"` 고정값
- **방법1 (권장)**: 컨트롤러에서 atoaTpc로 if-else 분기
- **방법2 (급할 때)**: JSP에서 `<c:set>` — template-main.jsp에서 "NH금융상품비교" 자동 추가되므로 뒤에 안 넣어도 됨
- **atoaTpc 분기**: 1=네이버, 2=카카오, 3=토스, 4=PASS, 7=NH인증

### [BUG] fcPopup.js SyntaxError — 해결
- **상태**: 해결
- **문제**: PT smartdev에서 공유하기 팝업(FCCOP0120P) 열 때 `Uncaught SyntaxError: Failed to execute 'appendChild'`
- **원인**: FCCOP0120P JSP의 initPage 함수에서 `}` 누락
- **해결**: `}` 추가
- **확인**: fcPopup.js 접근성 수정(fadeIn 콜백, aria-modal 등)은 문제없음

### 남은 할 일
- [ ] CBT 정상화 후 사업자 MCI 검증서버 테스트 (금요일)
- [x] MT 011 if문 제거 — 수정 완료 + alert 테스트 PASS + 기획서 확인
- [ ] 전자인증서 5개 pageTitle 적용
- [ ] 하드코딩 전부 원복 (커밋 전)
- [ ] 스크래핑 연결 후 MCI 4필드 실데이터 재테스트
- [ ] 배포리스트 엑셀 작성

<!-- 템플릿
### [카테고리] 제목
- **상태**: 해결 / 미해결 / 보류
- **문제**: 
- **원인**: 
- **해결**: 
- **배운 점**: 

카테고리: BUG, MCI, CBT, DB, CONFIG, TEST, BRIDGE, 기타
-->
