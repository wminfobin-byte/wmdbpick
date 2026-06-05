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

### 탭 구조
1. **분석** — 요일별/매체별 평균, 타임보드/스페셜 시간대별 분석
2. **차주 예상** — 요일별 예상 DB수, 타임보드/스페셜 스케줄, 계열별 배분, 1인 분배 예상
3. **공유** — 예상 결과 공유용 뷰

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
