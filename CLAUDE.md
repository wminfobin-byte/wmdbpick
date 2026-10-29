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
   - 근태 텍스트 변환 `wsParseVal`: 출근→1, 반차→0.5(연두), 오전/오후 결근→0.5(주황), 연차·휴가·경조·공가·장기공석→파랑, 대체휴무→노랑, 결근·퇴사→빨강
   - 집계: 평일 / 토요(토요일만) / 휴일(일요일+평일 공휴일). 그룹별·센터 블록은 1.0만 카운트, 센터별 합계는 0.5 포함 (원본 수식과 동일)
   - 붙여넣기 파서 `wsParsePaste`: 날짜(`N월 N일`/`N/N`) 줄마다 구간 분리 → `wsFindCenters`(센터명, 없으면 `WS_REGION_ALIAS` 충청·중부→대전·유성, 수도권·서울→당산·강동) → `wsExtractMgrs`(`매니저 : 이름`/`이름 매니저`/센터장 → `wsManagers` 칸, 수도권/중부는 센터로 판단) → `wsMatchNames`(동명이인은 체크 안 하고 경고)
   - 명단: 근태보고 `담당` 열 — 뇌새김(영어)→영어, 제2외국어→제2외국어, 톡이즈→톡이즈, 공란(팀장·매니저) 제외 (`wsGroupOf`). 센터 블록은 센터 안 그룹을 인원순으로 한 행씩
   - 엑셀 수식: 근무일정 A~C(센터·성명·그룹)만 값, 나머지 집계는 원본처럼 수식(주차 SUM, D=토요+일요, COUNTIFS/SUMIF로 근무일정 참조, 계=위 행 SUM). 평일/토요/휴일 합계 수식은 `wsTypeCols`로 그 달 달력에 맞춰 매달 새로 생성 (`wsSumF`가 연속 열을 범위로 묶음)
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
