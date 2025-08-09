## 목적

이 저장소는 아이트래킹 CSV 원시데이터로부터 Stimulus ID(자극)별 시선 이동 궤적을 간단한 폴리라인으로 시각화(PNG)하는 도구를 제공합니다. (배경 이미지 오버레이는 제외. 추후 UX에서 사용자 매핑으로 처리 권장)

## CSV 구조(필드 해석)

예시 헤더:

```
TimeStamp [us],Name,Record Date,Duration,Resolution X,Resolution Y,Monitor Size [inch],Version,Sampling Rate,Sample Count,Stimulus ID,Stimulus Version,Type,Key,Value,Sample Index,Filtered POR X,Filtered POR Y,Raw POR X,Raw POR Y,Left POR X,Left POR Y,Left Position X,Left Position Y,Left Position Z,Left Radius,Right POR X,Right POR Y,Right Position X,Right Position Y,Right Position Z,Right Radius,Fixation,Fixation Duration [ms],Fixation X,Fixation Y
```

- 핵심 메타
  - TimeStamp [us]: 마이크로초 단위 타임스탬프(세션 상대/절대, 기기별 상이)
  - Resolution X, Resolution Y: 표시 디스플레이 해상도(px)
  - Sampling Rate, Sample Count: 샘플링 빈도(Hz)와 전체 샘플 수
  - Stimulus ID: 자극 식별자(숫자/UUID 등)
  - Type: `event` | `data`
  - Key/Value: 이벤트 설명(`onset`, `timeout`, `Calibration` 등)

- 시선 좌표(픽셀)
  - Filtered POR X/Y: 필터 적용된 시선 좌표(Point of Regard)
  - Raw POR X/Y: 필터 미적용 원시 좌표

- 양안/3D/고정(Fixation) 관련(본 도구에서는 필수 아님)
  - Left/Right POR X/Y: 각 눈 기반 추정 좌표
  - Left/Right Position X/Y/Z: 눈의 3D 위치(보통 mm 추정)
  - Left/Right Radius: 동공 크기(기기 단위)
  - Fixation, Fixation Duration [ms], Fixation X/Y: 고정 검출 결과

주의: CSV에 비정상 따옴표(`"""text"""`)나 공백, 날짜 포맷 오류가 포함될 수 있어 이를 보정하는 전처리를 수행합니다.

## 단순 폴리라인 생성 로직(핵심)

입력: CSV 한 파일 → 출력: Stimulus ID별 1장의 PNG

1) 로딩/전처리
- 모든 컬럼을 문자열로 먼저 읽고, 필요한 컬럼만 숫자로 변환
- `Type`, `Stimulus ID`의 공백/과잉 따옴표 제거, `Type`은 소문자화
- 누락된 핵심 컬럼은 빈 컬럼으로 보완(견고성)

2) 그룹화
- `Stimulus ID`가 비어있지 않은 행만 유지
- `Stimulus ID`별로 그룹화

3) 샘플 선택/정렬
- 그룹 내 `Type == data` 행만 사용(이벤트 제외)
- 가능하면 `TimeStamp [us]` 기준 정렬

4) 좌표 선택
- 기본: `Filtered POR X/Y` 사용
- 필터 좌표가 비어있으면 `Raw POR X/Y`로 대체

5) 좌표 정제
- 좌표 없는 행 제거
- 그룹 내 중위수 해상도(`Resolution X/Y`) 추정(없으면 1920x1080)
- 화면 범위 밖 좌표 제거(0 ≤ x ≤ width, 0 ≤ y ≤ height)

6) 렌더링
- 화면 비율을 유지한 캔버스 생성
- Y축을 뒤집어(상단이 0), 화면 좌측-상단이 원점처럼 보이도록 설정
- 시선 이동을 시간 순서대로 선분으로 연결해 폴리라인 그리기
- 축 눈금/격자 숨기기, 파일명은 `gaze_<StimulusID>.png`

이 로직은 단순 궤적만 그리고, 배경 자극 이미지는 사용하지 않습니다. 추후 UX에서 Stimulus–이미지 매핑을 받아 오버레이할 수 있습니다.

## 사용 방법

### 의존성 설치
```
pip install pandas matplotlib numpy
```

### 실행 예시
```
python plot_gaze_polylines.py "2025-08-07 'test' export record 'joy'.csv"
```

옵션:
- `--out DIR`: 출력 폴더(기본 `gaze_polylines`)
- `--raw`: Raw POR 사용(필터 좌표 대신)

## 코드 개요(핵심 함수)

- `load_csv(path)`
  - CSV를 견고하게 읽고, 필수 컬럼 숫자 변환 및 `Type`/`Stimulus ID` 정리

- `pick_por_columns(df, prefer_filtered=True)`
  - 사용 가능한 좌표 컬럼을 결정(Filtered 선호, 없으면 Raw)

- `get_resolution(df_group)`
  - 그룹 내 해상도 중위수로 화면 크기 추정(없으면 1920x1080)

- `plot_polyline_for_group(group, stimulus_id, out_dir, prefer_filtered=True)`
  - `data` 행만 추출 → 타임스탬프 정렬 → 좌표 선택/정제 → 화면 비율/축 설정 → 폴리라인 저장

- `main()`
  - 인자 파싱 → CSV 로드 → `Stimulus ID`별 반복 처리 → PNG 저장

## 해석 팁

- 정규화 좌표가 필요하면: `x_norm = x / ResolutionX`, `y_norm = y / ResolutionY`
- 상대 시간(ms): `(TimeStamp_us - 첫값) / 1000`
- 고정(Fixation)만 분석하려면 해당 컬럼 필터링 후 별도 로직 추가 가능

## 한계/향후 확장

- 배경 이미지 오버레이: 현재 제외. 추후 UX에서 매핑 제공 시 오버레이 가능
- 고정점/사카드 표기: 폴리라인 위에 마커/색상으로 구분 확장 가능
- 화면 밖 좌표 처리: 현재 삭제. 시각적 디버그(점선 등) 추가 가능

