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
- [ ] CBT 테스트 시나리오 실행
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
