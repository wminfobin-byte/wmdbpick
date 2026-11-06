# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

WM DB 유입 예상 대시보드 — 영업관리팀의 차주 DB 유입량 예측을 위한 브라우저 기반 단일 파일 웹앱. CMS에서 추출한 엑셀 파일을 업로드하면 KPI, 차트, 예상 테이블을 자동 생성한다. 모든 데이터 처리는 클라이언트(브라우저)에서 수행되며 서버/백엔드 없음.

## Deployment

- **실서버**: GitHub Pages — https://wminfobin-byte.github.io/wmdbpick/
- **레포**: `wminfobin-byte/wmdbpick` (main 브랜치 push → 자동 배포)
- **wm_daily 레포와 별개** — wmdbpick 변경을 wm_daily에 push하지 말 것
- 로컬 테스트: `npx http-server -p 8081` 또는 브라우저에서 index.html 직접 열기

## Architecture

`index.html` 단일 파일 (CSS + HTML + JS 전부 인라인, ~2600줄). 빌드 스텝 없음.

### 외부 의존성 (CDN)
- **SheetJS (xlsx 0.18.5)** — 엑셀 파싱 (.xlsx/.xls)
- **Chart.js 4.4.1 + chartjs-plugin-datalabels** — 차트 시각화

### 데이터 저장
- **IndexedDB** (`db_forecast_idb`) — allData, schedules, attendanceData, tierConfig 등 영구 저장
- localStorage 마이그레이션 코드 존재 (레거시)

### 성능 관련 규칙
- **집계 결과 캐시** `_aggMemo`: `getGlobalDOWDateCounts`/`getBatchDOWDateCounts`/`getDOWAverageByType`/`getMediaDOWTable(ByPeriod)`/`getSmartMediaDOWTable(ByPeriod)`(키에 `workingDayToggle` 포함)/`getDowLangMediaByWeek`/`getLastInflowDateStr`. 원본은 `_raw_*`. **allData를 바꾸면 반드시 `invalidateAgg()` 호출** (캐시가 같이 비워짐). 결과에 영향을 주는 새 전역 상태를 추가하면 캐시 키에 넣을 것
- 공휴일 조회는 `KR_HOLI_SET.has()` (배열 indexOf 금지 — 레코드 순회 안에서 호출됨)
- **data.json 형식** `{"ts":"<lastSyncTime>","z":"<gzip base64>"}` — `ts`가 반드시 맨 앞. `fetchRemoteSyncTs()`가 Range 요청으로 앞 200바이트만 읽어 원격 시각을 확인하고, 원격이 더 새로울 때만 전체를 받는다. ts 없는 구형식이면 전체 다운로드로 대체

### 탭 구조
1. **분석** — 요일별/매체별 평균, 타임보드/스페셜 시간대별 분석
2. **차주 예상** — 요일별 예상 DB수, 타임보드/스페셜 스케줄, 계열별 배분, 1인 분배 예상
3. **공유** — 예상 결과 공유용 뷰
4. **근무현황** (로그인 전용) — 근태보고(`attendanceData`)로 원본 `뇌새김 n월 근무현황파일_종합` 엑셀의 종합현황·근무일정 시트를 재현. `ws*` 함수군 (index.html 하단)
   - 셀 모델(`wsModelSummary`/`wsModelSchedule`) 하나로 화면(`wsModelHTML`)과 엑셀(`wsToSheet`, xlsx-js-style+JSZip 지연 로드)을 같이 만든다
   - 근태 텍스트 변환 `wsParseVal`: 출근·지각(10분/20분/20분 초과)·조퇴→1, 반차→0.5(연두), 오전/오후 결근→0.5(주황), 연차·휴가·경조·공가·장기공석→파랑, 대체휴무→노랑, 결근·퇴사→빨강
   - 집계: 평일 / 토요(토요일만) / 휴일(일요일+평일 공휴일). 그룹별·센터 블록은 1.0만 카운트, 센터별 합계는 0.5 포함 (원본 수식과 동일)
   - 붙여넣기 파서 `wsParsePaste`: 날짜(`N월 N일`/`N/N`) 줄마다 구간 분리 → `wsFindCenters`(센터명, 없으면 `WS_REGION_ALIAS` 충청·중부→대전·유성, 수도권·서울→당산·강동) → `wsExtractMgrs`(`매니저 : 이름`/`이름 매니저`/센터장 → `wsManagers` 칸, 수도권/중부는 센터로 판단) → `wsMatchNames`(동명이인은 체크 안 하고 경고)
   - 명단: 근태보고 `담당` 열 — 뇌새김(영어)→영어, 제2외국어→제2외국어, 톡이즈→톡이즈, 공란(팀장·매니저) 제외 (`wsGroupOf`). 센터 블록은 센터 안 그룹을 인원순으로 한 행씩
   - 엑셀 수식: 근무일정 A~C(센터·성명·그룹)만 값, 나머지 집계는 원본처럼 수식(주차 SUM, D=토요+일요, COUNTIFS/SUMIF로 근무일정 참조, 계=위 행 SUM). 평일/토요/휴일 합계 수식은 `wsTypeCols`로 그 달 달력에 맞춰 매달 새로 생성 (`wsSumF`가 연속 열을 범위로 묶음)
   - 휴일 담당자 `WS_MGR_SLOTS`: 수도권 매니저·중부 매니저·센터장·분배 담당자 각 1명 (`wsManagers[ds][k]=[이름]`, 구버전 다중 슬롯은 첫 값 사용). 직급·센터 명부 = `WS_MGR_DIR_DEFAULT` + IDB `wsMgrDir`(화면 [담당자 명부]에서 수정, 동기화). 센터 미등록이면 근태파일에서 이름으로 찾음. `self`=본인 → 이름만 표시
   - 주말근무 보고 문구 `wsReportText`: 휴일별 그룹 인원(0.5도 1명)·총원·담당자를 메신저 보고 형식으로 생성, [문구 복사]
   - 휴일 근무 통계 `wsHsHTML` (구 `뇌새김 상담원 휴무.xlsx`의 토요근무/일요근무/평일공휴일출근/평일공휴일휴무 시트 N15:U43): 날짜(date 입력, 보관된 달의 평일 포함)를 골라 근무일정(특근=근태+`wsOverrides`) vs CMS 근태(원본)를 등급(A~D, `classifyStaffGroup`)·제2외국어·톡이즈 / 그룹 × 6개 센터(`WS_CENTERS_DEFAULT` 고정)로 집계, 숫자가 다르면 빨간 칸+이름 툴팁. 평일(공휴일 포함)은 출근/휴무(1.0이 아닌 사람, 퇴사 제외) 전환 — 일반 평일은 휴무가 기본(`wsHsPick`). 월말·월초용으로 달별 마지막 근태를 `wsAttArchive`(IDB·동기화, 최근 3개월)에 보관 → 날짜 목록이 보관된 달 전체
   - [출근자 리스트 다운로드] `wsHsDownloadWorkers`: 고른 날짜의 근무일정(특근) 출근자(평일·휴무 모드와 무관, 근무 통계 그룹 표와 같은 사람)를 `담당 DB | 센터 | 이름` 엑셀로. 담당 DB = 영어(근태 뇌새김영어)·제2외국어·톡이즈, 이름은 근태파일 원본 그대로(`(R_센터)이름`), 담당 → 센터(`WS_CENTERS_DEFAULT` 순) → 이름 순
   - [휴일 파일 준비 (3종 올리기)] `hwPrepUpload` (`hw*` 함수군): 주말 근무자용 엑셀을 올리면 고른 날짜 기준으로 채워 **같은 파일 이름**으로 다시 받는다. ExcelJS가 아니라 JSZip으로 원본 XML을 직접 고침(손대지 않은 시트·서식·수식·차트 보존, calcChain 삭제 + fullCalcOnLoad). 셀은 inlineStr, 공유 수식은 `hwParseRows`가 펼쳐서 `hwShiftF`로 행 이동
     - 시간대별 성과(`개인별`+`db수` 시트): db수·계약수 3행부터 삭제, 센터별·그룹별 B2 = 날짜 serial(서식 mm월 dd일), 그룹 시트 A~F = 그 달 근태파일 전체(담당 있는 사람, D = 영어/제2외국어/톡이즈), 개인별 E6~ = "▲ 센터" 줄 순서대로 센터 블록(출근자 + 여유 최대 3행 + ▲ 줄), 나머지 빈 수식 행. 수식은 기존 사람 행을 본보기로 행 이동
     - 휴무 개인별 성과(날짜 시트 누적): 마지막 시트를 본보기로 새 시트 `yymmdd`(B2 `  * m/d(요일) 휴일근무 개인별성과`, 센터 블록 + SUM 소계 + 총계 + K열 `총 n 건`). L열 오른쪽 표는 그대로 복사
     - 휴일근무 근태보고(`근태보고` 시트): 마지막 시트 복사 → 새 시트 `yymmdd`. 날짜, 총원 = 근태파일 전체 인원의 `(R_센터)` 기준(담당 공란 포함), 그룹별 신청 = 근무일정(특근) 출근, 출근 칸 비움, 특이사항 = `* 매니저 출근 : 센터 이름 | 센터 이름`(휴일 담당자 수도권·중부)
     - 2026-11-07 기준 사용자가 직접 만든 시트와 칸 단위로 일치 확인. 업무 엑셀은 `.gitignore`로 커밋 막음
   - 그날 기준 인원 `wsHist`(IDB·동기화, 달이 바뀌어도 유지): 근태 업로드마다 전체 인원(담당 무관)의 입사일·처음/마지막으로 보인 파일 날짜·마지막 근태 기록일(`act`, 미리 등록된 주말 출근 포함)·센터/담당/등급 변경 구간(`segs`)을 쌓는다. `wsHistAt(ds)` = 입사일 ≤ ds 이고 (최신 파일에 있음 또는 max(last,act) ≥ ds), 속성은 ds 시점 구간. 휴일 근무 통계의 모든 숫자와 **재적** 줄(이름의 `(R_센터)` 표시가 있고 담당이 있는 인원, 괄호 안 센터 기준. 담당 공란은 R_ 표시가 있어도 제외)이 이 기준
   - 퇴사자 유지 `wsRoster`(IDB·동기화): 같은 달 근태파일에 한 번이라도 있던 사람을 보관. 새 파일에서 빠지면 `wsAllPeople`이 명단에 남기고 이전 기록 유지 + 마지막 확인 파일 날짜 다음 날~현재 파일 날짜를 `퇴사`(빨강)로 채움. 달이 바뀌면 초기화. `attendanceData` 자체는 건드리지 않음(차주 예상 1인 분배 영향 없음)
   - 휴일 판정 `wsIsHoliday`: 토·일은 요일, 평일 공휴일은 IDB `wsHoliExtra`(수동 지정, true=휴일 추가/false=해제, 동기화 포함)가 우선, 없으면 `KR_HOLIDAYS`. 표 날짜 칸 클릭으로 지정/해제(`wsToggleHoliday`). 평일인데 근무자가 20% 미만인 지난 날은 임시공휴일 후보로 안내(`wsHoliCandidates`). 근무현황 탭에만 적용 (차주 예상은 `KR_HOLIDAYS`·`holidayClosed` 그대로)
   - IDB `wsManagers`(휴일 담당 매니저·센터장), `wsOverrides`(붙여넣기·클릭 수동 출근 변경, 근태 재업로드에도 유지), `wsCenters`(대상 센터, 기본 당산·강동·대전·유성·송내·인천) — GitHub 동기화 payload에 포함

## Key Data Model

### DB 엑셀 파일 컬럼 (0-indexed)
- G=6: customerKey, H=7: obName, I=8: distributeDate, J=9: inflowDate
- P=15: dbInfo (DB정보), T=19: mediaCode, U=20: package

### 데이터 필터링 (`shouldExclude`)
유입 산정 대상 판별 로직:
- 미디어코드 빈값 → 제외
- OB명에 `관리자` → 제외
- **P열 DB정보**: `IN`, `재콜` 포함 → 제외
- **P열 DB정보**: 공란 또는 `(재분배)` 정확히 일치하는 것만 산정 (그 외 값 → 제외)
- 미디어코드에 `callback`, `tv_b`, `_h_` → 제외

### 분류 체계
- **DB 유형** (mediaCode 기반): 영어, 일본어, 중국어, 스페인어, 프랑스어, 톡이즈
- **매체**: GFA, 카카오, 메타, GDN, 당근, 일반, 타임보드, 스페셜
- **타임보드/스페셜**: mediaCode에 `naver_tb`/`naver_sp` 포함 여부로 판별. 시간대는 미디어코드 끝 `숫자AM/PM` 패턴에서 파싱

### 스케줄 (차주 예상)
- 타임보드/스페셜 스케줄을 수동 등록
- **예상 DB수**: 직접 입력(수동) 또는 allData 평균 기반 자동계산. `manualDB > 0`이면 수동 모드
- 비중(%)대로 DB유형별 분배
- 5PM 이후 방송은 자동으로 다음날 분배에 포함 (`DIST_CUTOFF_HOUR=17`)

### 평균 계산 로직
- allData만 사용 (tbspData는 레거시, 사용하지 않음)
- 동일 날짜+시간에 2개 이상 패키지 동시 집행 시 해당 날짜+시간 레코드 제외
- 날짜별 수량 3건 이하인 날은 노이즈로 간주하여 제외 (전부 3 이하면 폴백)
- 해당 시간대에 데이터 없으면 전체 시간대 평균으로 폴백

### 주요 상수
- `WEEK_ORDER=[6,0,1,2,3,4,5]` — 토요일 시작 (분배 주기)
- `workingDayToggle=[true,false,true,true,true,true,true]` — 일요일 비분배 (요일 단위 수동 토글)
- `KR_HOLIDAYS` — 한국 공휴일 다년치(2024~2026). 매년 갱신 필요

### 주간 예상표 (양식) / 1인 분배 (`ys*` 함수군)
- 차주 예상 [주간 예상표 (양식)] 버튼 → `downloadYesangExcel()`: `yesang_template.xlsx`(원본 `N월 N주 계산 - 예상.xlsx`의 DB 예상안 시트 + D등급 블록)를 ExcelJS(CDN 지연 로드)로 열어 **입력 칸만** 채운다. 서식·수식은 양식 그대로. 파일명 = 월요일 기준 `N월 N주 계산 - 예상.xlsx`
- 양식 행 위치: A 5~11, B 12~18, C 19~25, **D 26~32**, 제2외 33~35, 총원 37 (원본 J26:AC41을 7행 내림). 뇌새김 타임보드(31행)는 17시 이후 방송을 다음날로 (DIST_CUTOFF_HOUR와 동일).
- 타임보드 표(B5:H28)에는 영어 수량만 넣고, 제2외국어 타임보드는 방송 요일·시간 칸을 초록색으로만 칠한다(수량은 분배일의 35행 오전TB = 전날 17시~당일 9시 방송 / 37행 오후TB = 10~16시). 양식 자체에는 고정 색이 없다. ExcelJS는 서식 객체를 여러 칸이 공유하므로 칸 서식을 바꿀 땐 `c.style=Object.assign({},c.style,{fill:...})`처럼 복사 후 변경할 것 양식을 바꾸면 `downloadYesangExcel`의 셀 주소도 같이 고칠 것
- 인원: 근무 통계의 근무일정(특근) `wsHsCalc().plan.g` — 토·일 = 출근 인원, 평일 = 휴무 인원, 기준인원 K = 첫 평일의 출근+휴무. 분배 인원 n = 토·일 출근 / 평일 K − 휴무 (엑셀 수식과 동일)
- 1인 분배 `ysAllocate` (2026년 7~10월 수기 예상표 15주 분석): 하루 1인 총량 A·B·C 동일, D = 평일 정상 근무일에만 70%(`GROUP_WEIGHT`), 휴일(토·일·공휴일 근무)은 A~D 모두 동일. 분배량 = 누적 예상 − 누적 분배(일요일·휴무일 유입과 반올림 오차는 다음 근무일로). 받는 계열 `YS_ELIG` 평일 A0~5/B1~5/C3~5/D4~5, 휴일(토·일·공휴일) A0~5/B1~5/C·D2~5. 평일은 학습 비중, 휴일은 범위 안 균등에서 출발해 IPF로 등급 총량·계열 예상을 동시에 맞춘 뒤 정수 반올림. 제2외국어는 상위/하위 비중 ÷ 인원
- 학습 `yesangLearn`(IDB·동기화): 계열별 분배 설정 > [예상 파일 올려 학습]으로 수기 수정한 예상 파일을 올리면 평일 등급별 계열 비중을 다시 계산 (`ysLearnFromSheet`: J열 `X등급` 블록 탐색, 평일·공휴일 아닌 날만). 없으면 `YS_SHARE_DEFAULT`
- 화면 1인 분배·공유 탭·통합 엑셀 '등급별 1인 분배'도 같은 `lastPredData.ys`를 쓴다. 구 `tierConfig.groups[].basis`(등급별 계열 기준 1인당)는 더 이상 쓰지 않음 (UI 숨김)

### 평일 공휴일 처리 (차주 예상)
- `holidayClosed` — 날짜 단위 휴무 override (`{'YYYY-MM-DD':true}`). IndexedDB `holidayClosed`에 영구 저장
- **평일(월~금) 공휴일은 기본 영업(ON)**, `holidayClosed`에 등록된 날만 휴무. 토/일 공휴일은 기존 `workingDayToggle` 따름. 분배일 토글 UI는 평일 공휴일에 한해 날짜 단위(`holidayClosed`)를 토글
- **휴무 평일 공휴일** → 비분배·이월(캐리오버). 단, base가 이미 `'H'`(그 요일 평일 예상의 50%)이므로 `applyCarryover`의 연속 OFF 50% 추가 감소는 **적용하지 않음**(`isWeekdayHoliday` 제외) → 휴무 공휴일은 단독·연속 무관하게 그 요일 50%를 이월. (주말 OFF만 연속 시 50% 추가 감소)
- 헬퍼: `isPredHoliday(i)`, `isWeekdayHoliday(i)`(월~금 공휴일), `isWorkDay(i)`(실제 분배 여부), `effDow(i)`(평균 조회용 실효 요일 키 — 평일 공휴일은 `'H'+요일`(예: `'H3'`) 반환)
- 차주 예상 계산은 `workingDayToggle[i]` 직접 참조 대신 `isWorkDay(i)`, DOW 평균 조회는 `effDow(i)`를 사용할 것 (차주 예상·캐리오버·수동조정·공유·엑셀 공통)
- 타임보드/스페셜 스케줄은 공휴일에도 집행 가능(영향 없음)

### 평일 공휴일 DB량 (해당일 평일 예상의 50%)
- 평일 공휴일 영업일의 DB량 = **그 요일 평일 예상 × 0.5** (단순 고정. 과거엔 토요일 수준·학습·비중 방식을 거쳤으나 최종적으로 50% 고정으로 정리)
- `recInflowHoli(d)`/`recBatchHoli(d)` — 레코드가 평일 공휴일 소속인지 판정. `ensureAgg()` 및 플레인 DOW 테이블(`getDOWAverageByType`/`getMediaDOWTable*`/`get*DOWDateCounts`)에서 평일 공휴일 레코드를 **요일 평균에서 제외** (요일 평균이 공휴일에 오염되지 않도록)
- 스마트 테이블의 **`'H1'`~`'H5'` 컬럼**(요일별) = `그 요일 평일 예측 × 0.5`
- `effDow(i)`가 평일 공휴일에 `'H'+요일`(예: 수요일 공휴일 → `'H3'`)을 반환 → 4개 소비처(`getMediaDOWTotal`/`buildPredSection`/공유/엑셀)가 `tAM[m]['H3']` 등을 조회
- 차주에 평일 공휴일이 있으면 분배일 토글 영역에 50% 적용 안내 노트 표시

## Language

프로젝트 오너는 한국어 사용자. 한국어로 소통할 것.
