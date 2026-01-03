# 🏃 디지털 휴먼 트윈 러닝 시뮬레이터 - 최종 설계안

## 📋 프로젝트 개요

### 한 줄 요약
Apple Watch 러닝 기록을 분석하여 **디지털 휴먼 트윈**을 생성하고, 사용자가 입력한 1~4주 루틴을 가상으로 실행하여 성장량과 훈련 부하를 예측하는 macOS 데스크톱 앱

### 프로젝트 목적
- **공모전 제출**: 디지털 휴먼 트윈 프로젝트
- **시연 환경**: 로컬 MacBook (M1 Pro)
- **개발 기간**: 2주 (14일)

---

## 🎯 목표와 제약사항

### 1차 MVP 성공 조건
1. ✅ macOS Flutter 데스크톱 앱 실행
2. ✅ Health Export ZIP 업로드 → 러닝 워크아웃 + HR 시계열 시각화
3. ✅ 최소 3~5개 워크아웃으로 **디지털 트윈 생성**
4. ✅ 1~4주 루틴 입력 → 시뮬레이션 → 성장량 + 부하 경고 출력
5. ✅ 3가지 시연 시나리오 준비 (초보/중급/고급 러너)

### 제약사항
- 로컬 전용 (네트워크 최소화)
- 러닝 종목만 지원
- 실시간 수집 없음 (Health Export 기반)
- 경량 구현 (CNN+ODE 제외)

---

## 🏗️ 시스템 아키텍처

### 구성요소
```
┌─────────────────────────────────────────┐
│         Flutter Desktop (macOS)         │
│  - 트윈 생성 UI                          │
│  - 워크아웃 시각화 (HR 그래프)            │
│  - 루틴 입력 인터페이스                   │
│  - 시뮬레이션 결과 대시보드               │
└───────────────┬─────────────────────────┘
                │ REST API (localhost)
┌───────────────▼─────────────────────────┐
│        FastAPI Backend (Python)         │
│  - Health Export ZIP 파싱               │
│  - 10초 리샘플링 전처리                  │
│  - 디지털 트윈 생성 (회귀 모델)           │
│  - Banister 시뮬레이션                   │
│  - TRIMP 계산                           │
└───────────────┬─────────────────────────┘
                │
┌───────────────▼─────────────────────────┐
│           SQLite (로컬 DB)              │
│  - workouts (워크아웃 메타데이터)         │
│  - hr_timeseries (10초 시계열)          │
│  - digital_twins (트윈 프로필)          │
│  - plans (루틴 계획)                    │
│  - simulation_results (시뮬레이션 결과)  │
└─────────────────────────────────────────┘
```

---

## 🛠️ 기술 스택

### 백엔드
- **FastAPI** (Python 3.10+)
- **SQLite** (로컬 DB)
- **scikit-learn** (선형 회귀)
- **NumPy/Pandas** (데이터 처리)
- **xml.etree.ElementTree** (Health Export 파싱)

### 프론트엔드
- **Flutter** (Desktop - macOS)
- **fl_chart** (HR 그래프)
- **http** (REST API 통신)
- **file_picker** (ZIP 업로드)

### 모델/알고리즘
- **선형 회귀** (페이스 → HR 예측)
- **Banister Fitness-Fatigue Model** (적응 시뮬레이션)
- **Zone-Based TRIMP** (훈련 부하 계산)

---

## 📊 데이터 파이프라인

### 1. Health Export ZIP 파싱

#### XML 구조
```xml
<!-- 워크아웃 -->
<Workout workoutActivityType="HKWorkoutActivityTypeRunning"
         duration="45.5" durationUnit="min"
         totalDistance="8.2" totalDistanceUnit="km"
         startDate="2025-01-01 08:00:00"
         endDate="2025-01-01 08:45:30"/>

<!-- 심박수 기록 -->
<Record type="HKQuantityTypeIdentifierHeartRate"
        startDate="2025-01-01 08:05:20"
        value="145" unit="count/min"/>
```

#### 파싱 로직
```python
import xml.etree.ElementTree as ET
from datetime import datetime

def parse_health_export(zip_path):
    """Health Export ZIP 파싱"""

    # 1. ZIP 압축 해제
    with zipfile.ZipFile(zip_path) as z:
        z.extract('apple_health_export/export.xml', '/tmp')

    # 2. XML 파싱 (스트리밍)
    workouts = []
    hr_records = []

    for event, elem in ET.iterparse('/tmp/export.xml'):
        if elem.tag == 'Workout' and \
           elem.get('workoutActivityType') == 'HKWorkoutActivityTypeRunning':
            workouts.append({
                'start': elem.get('startDate'),
                'end': elem.get('endDate'),
                'duration': float(elem.get('duration', 0)),
                'distance': float(elem.get('totalDistance', 0))
            })
            elem.clear()

        elif elem.tag == 'Record' and \
             elem.get('type') == 'HKQuantityTypeIdentifierHeartRate':
            hr_records.append({
                'time': elem.get('startDate'),
                'hr': float(elem.get('value'))
            })
            elem.clear()

    return workouts, hr_records
```

### 2. 10초 리샘플링

```python
def resample_to_10s(workout, hr_records):
    """HR 시계열을 10초 간격으로 리샘플링"""

    # 워크아웃 시간 범위에 해당하는 HR만 추출
    workout_hrs = [
        hr for hr in hr_records
        if workout['start'] <= hr['time'] <= workout['end']
    ]

    # pandas로 10초 리샘플링
    df = pd.DataFrame(workout_hrs)
    df['time'] = pd.to_datetime(df['time'])
    df.set_index('time', inplace=True)

    # 10초 간격으로 리샘플 (선형 보간)
    resampled = df.resample('10S').mean().interpolate(method='linear')

    # 이상치 제거 (45-215 bpm)
    resampled = resampled[(resampled['hr'] >= 45) & (resampled['hr'] <= 215)]

    return resampled
```

### 3. 데이터 품질 검증

```python
def validate_workout(workout, hr_timeseries):
    """워크아웃 품질 검사"""

    # 최소 길이 (15분)
    if workout['duration'] < 15:
        return False, "너무 짧음"

    # HR 데이터 충분성 (50% 이상)
    expected_points = workout['duration'] * 60 / 10
    if len(hr_timeseries) < expected_points * 0.5:
        return False, "HR 데이터 부족"

    # 거리 타당성 (최소 1km)
    if workout['distance'] < 1.0:
        return False, "거리 부족"

    return True, "OK"
```

---

## 🤖 디지털 트윈 모델

### 1. 트윈 프로필 구조

```python
digital_twin = {
    # 기초 프로필
    "user_id": "user_123",
    "created_at": "2025-01-03",

    # 생리학 파라미터
    "resting_hr": 62,           # bpm
    "max_hr": 188,              # bpm (추정)
    "aerobic_threshold": 165,   # bpm

    # 페이스-HR 관계 (선형 회귀)
    "hr_model": {
        "slope": 25.3,          # HR 증가율 (per min/km)
        "intercept": 85.2,      # 기본 HR
        "r_squared": 0.94       # 정확도
    },

    # 피트니스 상태
    "current_fitness": 72,      # /100
    "estimated_vo2max": 48,     # ml/kg/min

    # 적응 파라미터 (Banister)
    "tau_fitness": 50,          # 일 (피트니스 감쇠)
    "tau_fatigue": 10,          # 일 (피로 감쇠)
    "k_fatigue": 1.8,           # 피로 가중치

    # 회복 능력
    "recovery_rate": "평균 이상"
}
```

### 2. 트윈 생성 로직

```python
from sklearn.linear_model import LinearRegression
import numpy as np

class DigitalTwin:
    def __init__(self):
        self.hr_model = LinearRegression()
        self.profile = {}

    def create_from_workouts(self, workouts):
        """워크아웃 데이터로 트윈 생성"""

        # Step 1: 기초 통계
        all_hr = []
        for w in workouts:
            all_hr.extend(w['hr_timeseries'])

        self.profile['resting_hr'] = np.percentile(all_hr, 5)
        self.profile['max_hr'] = 220 - 30  # TODO: 사용자 입력

        # Step 2: 페이스-HR 관계 학습
        X = []  # 페이스 (분/km)
        y = []  # 평균 HR

        for w in workouts:
            pace = w['duration'] / w['distance']  # 분/km
            avg_hr = np.mean(w['hr_timeseries'])
            X.append([pace])
            y.append(avg_hr)

        X = np.array(X)
        y = np.array(y)

        self.hr_model.fit(X, y)
        r2 = self.hr_model.score(X, y)

        self.profile['hr_model'] = {
            'slope': self.hr_model.coef_[0],
            'intercept': self.hr_model.intercept_,
            'r_squared': r2
        }

        # Step 3: 피트니스 레벨 추정
        # 6:00/km 페이스에서의 HR이 낮을수록 피트니스 높음
        baseline_hr = self.predict_hr(6.0)
        fitness = 100 - (baseline_hr - 140) * 2
        self.profile['current_fitness'] = np.clip(fitness, 0, 100)

        # Step 4: VO2max 추정 (간단 공식)
        # VO2max ≈ 15 × (max_hr / resting_hr)
        vo2max = 15 * (self.profile['max_hr'] / self.profile['resting_hr'])
        self.profile['estimated_vo2max'] = round(vo2max, 1)

        # Step 5: Banister 파라미터 (기본값)
        self.profile['tau_fitness'] = 50
        self.profile['tau_fatigue'] = 10
        self.profile['k_fatigue'] = 1.8

        return self.profile

    def predict_hr(self, pace, fatigue_level=0):
        """페이스로 HR 예측 (피로도 반영)"""
        base_hr = self.hr_model.predict([[pace]])[0]
        fatigue_penalty = fatigue_level * 5  # 피로 10당 5bpm 증가
        return base_hr + fatigue_penalty
```

### 3. TRIMP 계산 (Zone 기반)

```python
def calculate_trimp(workout, twin):
    """Zone 기반 TRIMP 계산"""

    # HR 존 정의 (% max HR)
    zones = {
        1: (50, 60, 1.0),   # 회복 존
        2: (60, 70, 2.0),   # 유산소 존
        3: (70, 80, 3.0),   # 템포 존
        4: (80, 90, 4.0),   # 역치 존
        5: (90, 100, 5.0)   # VO2max 존
    }

    max_hr = twin.profile['max_hr']
    trimp = 0

    # 10초 시계열 각 포인트 처리
    for hr in workout['hr_timeseries']:
        hr_percent = (hr / max_hr) * 100

        # 해당 존 찾기
        for zone_num, (min_pct, max_pct, weight) in zones.items():
            if min_pct <= hr_percent < max_pct:
                trimp += weight * (10 / 60)  # 10초 = 1/6분
                break

    return trimp
```

### 4. Banister 시뮬레이션

```python
def simulate_training(twin, routine_weeks):
    """Banister Fitness-Fatigue 모델로 시뮬레이션"""

    fitness = twin.profile['current_fitness']
    fatigue = 0
    results = []

    tau_g = twin.profile['tau_fitness']  # 피트니스 감쇠 시간
    tau_h = twin.profile['tau_fatigue']  # 피로 감쇠 시간
    k = twin.profile['k_fatigue']         # 피로 가중치

    day = 0
    for week_num, week_workouts in enumerate(routine_weeks):
        week_trimp = 0

        for workout in week_workouts:
            day += 1

            # 오늘 운동이 있으면
            if workout:
                trimp = calculate_trimp_from_plan(workout, twin)
                week_trimp += trimp
            else:
                trimp = 0

            # Banister 수식
            # fitness(t) = fitness(t-1) × e^(-1/τ_g) + w(t)
            # fatigue(t) = fatigue(t-1) × e^(-1/τ_h) + w(t)
            fitness = fitness * np.exp(-1/tau_g) + trimp * 0.05
            fatigue = fatigue * np.exp(-1/tau_h) + trimp * 0.15

            # 컨디션 = 피트니스 - 피로
            condition = fitness - k * fatigue

            if day % 7 == 0:  # 주간 기록
                results.append({
                    'week': week_num + 1,
                    'fitness': round(fitness, 1),
                    'fatigue': round(fatigue, 1),
                    'condition': round(condition, 1),
                    'weekly_trimp': round(week_trimp, 1)
                })

    # 성장량 계산
    initial_hr_at_6min = twin.predict_hr(6.0)
    fitness_gain = fitness - twin.profile['current_fitness']
    final_hr_at_6min = initial_hr_at_6min - (fitness_gain * 0.15)

    return {
        'weekly_progress': results,
        'improvement': {
            'pace': '6:00/km',
            'hr_before': round(initial_hr_at_6min, 1),
            'hr_after': round(final_hr_at_6min, 1),
            'delta_bpm': round(initial_hr_at_6min - final_hr_at_6min, 1),
            'fitness_gain': round(fitness_gain, 1)
        },
        'warnings': generate_warnings(results)
    }

def generate_warnings(weekly_results):
    """과부하 경고 생성"""
    warnings = []

    for i in range(1, len(weekly_results)):
        prev_trimp = weekly_results[i-1]['weekly_trimp']
        curr_trimp = weekly_results[i]['weekly_trimp']

        # 10% 룰
        if curr_trimp > prev_trimp * 1.1:
            warnings.append({
                'week': i + 1,
                'type': 'overload',
                'message': f'Week {i+1}: 주간 부하가 10% 이상 증가 ({prev_trimp:.0f}→{curr_trimp:.0f}). 부상 위험!'
            })

        # 절대 부하
        if weekly_results[i]['fatigue'] > 80:
            warnings.append({
                'week': i + 1,
                'type': 'high_fatigue',
                'message': f'Week {i+1}: 피로도 과다 ({weekly_results[i]["fatigue"]:.0f}/100). 회복 주 권장!'
            })

    return warnings
```

---

## 🎨 핵심 기능

### Feature 1: 디지털 트윈 생성

**입력:** 3~5개 워크아웃 데이터
**출력:** 개인화된 트윈 프로필

**UI 연출:**
```
┌────────────────────────────────────┐
│  🔍 데이터 분석 중...              │
│  ━━━━━━━━━━━━━━━━━━━━ 100%        │
│                                    │
│  ✓ 5개 러닝 워크아웃 발견           │
│  ✓ 평균 페이스: 5:45/km            │
│  ✓ 평균 심박수: 158 bpm            │
│                                    │
│  🧬 생리학 프로필 생성 중...        │
│  ━━━━━━━━━━━━░░░░░░░░ 60%         │
│                                    │
│  분석 내용:                        │
│  • 안정시 HR: 62 bpm              │
│  • 최대 HR: 188 bpm (추정)        │
│  • 유산소 역치: 165 bpm           │
│                                    │
│  ⚙️ 페이스-심박 관계 학습...       │
│  ━━━━━━━━━━━━━━━━━━░░ 90%        │
│                                    │
│  6:30/km → 145 bpm ✓              │
│  6:00/km → 158 bpm ✓              │
│  5:30/km → 172 bpm ✓              │
│                                    │
│  정확도: R² = 0.94 (매우 높음!)    │
│                                    │
│  🎉 디지털 트윈 생성 완료!         │
│                                    │
│  👤 당신의 트윈 프로필              │
│  ├─ 현재 피트니스: 72/100         │
│  ├─ 추정 VO2max: 48 ml/kg/min     │
│  └─ 회복 능력: 평균 이상           │
│                                    │
│        [시뮬레이션 시작하기]        │
└────────────────────────────────────┘
```

### Feature 2: 가상 훈련 시뮬레이션

**입력:** 1~4주 루틴
**출력:** 주차별 성장 + 경고

**UI:**
```
┌────────────────────────────────────┐
│  📅 훈련 계획 입력                 │
│                                    │
│  주차: [1주] [2주] [3주] [4주] ✓  │
│                                    │
│  Week 1:                           │
│  월: Easy 5km @ 6:00/km           │
│  화: 휴식                          │
│  수: Tempo 3km @ 5:30/km          │
│  목: 휴식                          │
│  금: Easy 5km @ 6:00/km           │
│  토: 휴식                          │
│  일: Long 10km @ 6:30/km          │
│                                    │
│  [Week 2~4도 동일하게 반복]       │
│                                    │
│     [시뮬레이션 실행] 버튼          │
└────────────────────────────────────┘

실행 후:

┌────────────────────────────────────┐
│  🤖 트윈이 훈련 중...              │
│                                    │
│  Week 1: ████░░░░░░ 부하 증가     │
│          피트니스 72 → 74          │
│          피로도 0 → 25             │
│                                    │
│  Week 2: ██████░░░░ 적응 중       │
│          피트니스 74 → 77          │
│          피로도 25 → 30            │
│                                    │
│  Week 3: ████████░░ 성장!         │
│          피트니스 77 → 82          │
│          피로도 30 → 35            │
│          ⚠️ 주간 부하 +15% 경고   │
│                                    │
│  Week 4: ██████░░░░ 회복 주       │
│          피트니스 82 → 80          │
│          피로도 35 → 20            │
│                                    │
│  📈 4주 후 예상 성장량:            │
│                                    │
│  6:00/km 페이스에서                │
│  HR: 158 bpm → 150 bpm (-8 bpm)  │
│                                    │
│  피트니스: 72 → 80 (+8점)         │
│                                    │
│  💡 인사이트:                      │
│  Week 3 부하를 10% 줄이면          │
│  부상 위험 없이 동일 성장 가능      │
└────────────────────────────────────┘
```

### Feature 3: 시각적 비교

**그래프:**
```
HR @ 6:00/km 페이스
│
│ 160 ┤     ●  현재
│ 155 ┤     │
│ 150 ┤     │    ●  4주 후
│ 145 ┤     │    │
│ 140 ┤     ▼    ▼
│     └──────────────
     Week 0  Week 4

피트니스/피로 추이
│
│100 ┤
│ 80 ┤  ╱────────  피트니스
│ 60 ┤ ╱
│ 40 ┤╱  ╱╲╱╲     피로
│ 20 ┤  ╱    ╲
│  0 ┤─────────────
     W1 W2 W3 W4
```

---

## 🌐 API 설계

### 엔드포인트

#### 1. Import
```http
POST /api/import/health-zip
Content-Type: multipart/form-data

Request:
- file: export.zip

Response:
{
  "import_id": "imp_123",
  "workouts_found": 12,
  "workouts_valid": 10,
  "warnings": [
    "2개 워크아웃 제외 (HR 데이터 부족)"
  ]
}
```

#### 2. Workouts
```http
GET /api/workouts?limit=20&offset=0

Response:
{
  "total": 10,
  "workouts": [
    {
      "id": "w_001",
      "date": "2025-01-01",
      "duration_min": 45,
      "distance_km": 8.2,
      "avg_pace": "5:30/km",
      "avg_hr": 158,
      "quality": "high"
    },
    ...
  ]
}
```

```http
GET /api/workouts/{id}

Response:
{
  "id": "w_001",
  "date": "2025-01-01",
  "duration_min": 45,
  "distance_km": 8.2,
  "avg_pace": "5:30/km",
  "avg_hr": 158,
  "max_hr": 178,
  "hr_timeseries": [
    {"time": "08:00:00", "hr": 120},
    {"time": "08:00:10", "hr": 125},
    ...
  ],
  "trimp": 185.3
}
```

#### 3. Digital Twin
```http
POST /api/twin/create

Request:
{
  "workout_ids": ["w_001", "w_002", "w_003", "w_004", "w_005"],
  "user_age": 30,
  "user_max_hr": 190  // optional
}

Response:
{
  "twin_id": "twin_user_123",
  "profile": {
    "resting_hr": 62,
    "max_hr": 188,
    "current_fitness": 72,
    "estimated_vo2max": 48,
    "hr_model": {
      "r_squared": 0.94
    }
  },
  "status": "ready"
}
```

#### 4. Simulation
```http
POST /api/simulate

Request:
{
  "twin_id": "twin_user_123",
  "weeks": 4,
  "routine": [
    {
      "week": 1,
      "workouts": [
        {"day": "mon", "type": "easy", "distance_km": 5, "pace": "6:00"},
        {"day": "wed", "type": "tempo", "distance_km": 3, "pace": "5:30"},
        {"day": "fri", "type": "easy", "distance_km": 5, "pace": "6:00"},
        {"day": "sun", "type": "long", "distance_km": 10, "pace": "6:30"}
      ]
    },
    // week 2-4 ...
  ]
}

Response:
{
  "simulation_id": "sim_789",
  "weekly_progress": [
    {
      "week": 1,
      "fitness": 74,
      "fatigue": 25,
      "condition": 85,
      "weekly_trimp": 520
    },
    ...
  ],
  "improvement": {
    "pace": "6:00/km",
    "hr_before": 158,
    "hr_after": 150,
    "delta_bpm": -8,
    "fitness_gain": 8
  },
  "warnings": [
    {
      "week": 3,
      "type": "overload",
      "message": "주간 부하 15% 증가, 조정 권장"
    }
  ]
}
```

---

## 📱 Flutter UI 설계

### 화면 구조
```
main.dart
├── screens/
│   ├── import_screen.dart      (ZIP 업로드)
│   ├── workouts_screen.dart    (워크아웃 리스트)
│   ├── workout_detail_screen.dart (HR 그래프)
│   ├── twin_creation_screen.dart  (트윈 생성)
│   ├── plan_input_screen.dart     (루틴 입력)
│   └── simulation_result_screen.dart (결과)
├── widgets/
│   ├── hr_chart.dart           (HR 시계열 그래프)
│   ├── twin_profile_card.dart  (프로필 카드)
│   ├── progress_chart.dart     (피트니스/피로 그래프)
│   └── workout_card.dart       (워크아웃 카드)
└── services/
    └── api_service.dart        (REST API 통신)
```

### 주요 위젯

#### HR 그래프 (fl_chart)
```dart
import 'package:fl_chart/fl_chart.dart';

class HRChart extends StatelessWidget {
  final List<HRPoint> data;

  @override
  Widget build(BuildContext context) {
    return LineChart(
      LineChartData(
        lineBarsData: [
          LineChartBarData(
            spots: data.map((p) =>
              FlSpot(p.timeSeconds.toDouble(), p.hr.toDouble())
            ).toList(),
            color: Colors.red,
            dotData: FlDotData(show: false),
          )
        ],
        titlesData: FlTitlesData(
          bottomTitles: AxisTitles(
            sideTitles: SideTitles(
              showTitles: true,
              getTitlesWidget: (value, meta) {
                int minutes = value.toInt() ~/ 60;
                return Text('${minutes}min');
              }
            )
          )
        )
      )
    );
  }
}
```

---

## 📅 개발 단계 (2주)

### Week 1: 백엔드 + 기본 기능

**Day 1-2: 데이터 준비**
```
□ Health Export 샘플 받기 (iPhone)
□ XML 파싱 스크립트 작성
  - 러닝 워크아웃 추출
  - HR 기록 추출
  - 워크아웃별 HR 매칭
□ 10초 리샘플링 구현
□ 샘플 데이터 3세트 준비 (초보/중급/고급)
```

**Day 3-4: FastAPI + DB**
```
□ FastAPI 프로젝트 초기화
□ SQLite 스키마 설계
  CREATE TABLE workouts (...)
  CREATE TABLE hr_timeseries (...)
  CREATE TABLE digital_twins (...)
  CREATE TABLE simulation_results (...)
□ POST /import/health-zip 구현
□ GET /workouts 구현
```

**Day 5-6: 디지털 트윈 로직**
```
□ DigitalTwin 클래스 구현
  - create_from_workouts()
  - predict_hr()
□ 선형 회귀 학습 (scikit-learn)
□ POST /twin/create API
□ 테스트 (3-5개 워크아웃으로 트윈 생성)
```

**Day 7: 시뮬레이션 엔진**
```
□ TRIMP 계산 구현 (Zone 기반)
□ Banister 모델 구현
  - simulate_training()
  - generate_warnings()
□ POST /simulate API
□ 백엔드 통합 테스트
```

### Week 2: Flutter UI + 시연 준비

**Day 8-9: Flutter 기본 구조**
```
□ Flutter Desktop 프로젝트 생성
□ API Service 클래스
□ Import 화면
  - file_picker로 ZIP 선택
  - 업로드 진행 표시
□ Workouts 리스트 화면
```

**Day 10-11: 트윈 생성 UI**
```
□ Twin Creation 화면
  - 워크아웃 선택 (최소 3개)
  - "생성 중" 애니메이션
  - 프로필 표시 (카드 UI)
□ HR 그래프 위젯 (fl_chart)
□ Workout 상세 화면
```

**Day 12-13: 시뮬레이션 UI**
```
□ Plan Input 화면
  - 주차 선택 (1~4주)
  - 요일별 루틴 입력
  - 템플릿 제공 (Easy/Tempo/Long)
□ Simulation Result 화면
  - 주차별 진행 표시
  - 피트니스/피로 그래프
  - 성장량 요약
  - 경고 메시지
```

**Day 14: 시연 준비**
```
□ 3가지 시나리오 시연 데이터 준비
□ 발표 자료 작성
□ 데모 영상 촬영 (2-3분)
□ 버그 수정 및 마무리
```

---

## 🏆 공모전 어필 포인트

### 차별화 요소

#### 1. 과학적 근거
```
"단순 AI가 아닌, 검증된 스포츠과학 모델"

✅ Banister Fitness-Fatigue (1975~)
   - 올림픽 선수 훈련에 사용

✅ TRIMP (Training Impulse)
   - 스포츠과학 표준 지표

✅ Apple ML Research 참고
   - 세계 최고 수준 연구 기반
```

#### 2. 개인화
```
"당신만의 디지털 트윈"

- 일반 앱: 평균값 기반 조언
- 우리 앱: 개인 데이터 학습

예시:
사용자 A: 6:00/km → 150 bpm
사용자 B: 6:00/km → 170 bpm
→ 같은 페이스, 다른 부하!
```

#### 3. 예측 가능성
```
"미리 해보고 결정하기"

Before: "일단 훈련해보고 결과 확인"
After:  "트윈으로 미리 시뮬레이션"

→ 부상 예방, 동기부여, 효율적 계획
```

#### 4. 접근성
```
"최소 3개 워크아웃만 있으면 시작"

- 진입장벽 낮음
- 데이터 적어도 작동
- 로컬 전용 (개인정보 안전)
```

### 기술 스택 설명

**"왜 이 기술들을 선택했나?"**

| 기술 | 이유 |
|------|------|
| Flutter Desktop | 크로스플랫폼, 네이티브 성능, 빠른 개발 |
| FastAPI | 경량, 빠름, 타입 안전, 문서 자동 생성 |
| SQLite | 로컬 전용, 설정 불필요, 개인정보 안전 |
| scikit-learn | 표준 ML 라이브러리, 안정적, 경량 |
| 회귀 + Banister | 해석 가능, 스포츠과학 검증, 적은 데이터로 작동 |

---

## 🎬 시연 시나리오

### 시나리오 A: 초보 러너
```
페르소나: 러닝 시작 2주차, 5km 완주 목표
데이터: 3개 워크아웃 (Easy 위주)

1. Health Export 업로드
   "3개 워크아웃 발견"

2. 트윈 생성
   피트니스: 45/100
   "초급 수준, 성장 잠재력 높음"

3. 4주 루틴 입력
   주 3회, Easy 3-5km

4. 시뮬레이션
   → "4주 후 5km 완주 가능!"
   → Week 2 경고: "휴식일 부족, 추가 권장"

5. 결과
   피트니스 45 → 58 (+13)
   5km 페이스 7:00 → 6:30 (30초 개선)
```

### 시나리오 B: 중급 러너 (하프마라톤 준비)
```
페르소나: 하프 1:45 목표
데이터: 12개 워크아웃

1. 트윈 생성
   피트니스: 78/100
   VO2max: 52 ml/kg/min

2. 대회 전 4주 루틴
   - Week 1-2: 빌드업 (템포+인터벌)
   - Week 3: 피크 (최대 부하)
   - Week 4: 테이퍼링 (회복)

3. 시뮬레이션
   → "목표 기록 1:45 달성 가능!"
   → 최적 타이밍: Week 3 → Week 4 후 대회

4. 인사이트
   "Week 3 Long Run을 10km → 8km로 줄이면
    피로도 5% 감소, 동일 성장 유지"
```

### 시나리오 C: 부상 복귀
```
페르소나: 3개월 부상 휴식 후 복귀
데이터: 복귀 후 5개 워크아웃

1. 트윈 생성
   피트니스: 55/100
   "부상 전 대비 30% 감소 추정"

2. 안전 복귀 루틴 (10% 룰 적용)
   Week 1: 주 3회, 3km Easy
   Week 2: 주 3회, 4km Easy (+10%)
   ...

3. 시뮬레이션
   → 주간 부하 증가율 모니터링
   → ⚠️ Week 3: "10% 초과, 조정 필요"

4. 조정 제안
   "Week 3 거리를 1km 줄이면
    안전 범위 내 유지"
```

---

## ⚠️ 리스크 관리

### 기술적 리스크

| 리스크 | 확률 | 대응 방안 |
|--------|------|-----------|
| Health Export 형태 예상과 다름 | 30% | 샘플 먼저 분석, CSV 수동 입력 지원 |
| 10초 리샘플링 실패 (결측 많음) | 20% | 품질 낮은 워크아웃 제외, 경고 표시 |
| 회귀 정확도 낮음 (R² < 0.7) | 15% | 최소 워크아웃 수 증가 (5→7개) |
| Flutter Desktop M1 이슈 | 10% | 공식 문서 따라 설정, 커뮤니티 참고 |
| 시뮬레이션 결과 비현실적 | 25% | Banister 파라미터 튜닝, 상한선 설정 |

### 일정 리스크

**버퍼 전략:**
- Day 7: 백엔드 버퍼 (예비일)
- Day 14: 전체 버퍼 (마무리)
- 핵심 기능 먼저, 부가 기능 후순위

**우선순위:**
1. P0 (필수): 트윈 생성, 시뮬레이션, 결과 표시
2. P1 (중요): HR 그래프, 경고 메시지
3. P2 (선택): 템플릿, 여러 시나리오, 애니메이션

---

## 📚 참고 자료

### 학술 논문
- Apple ML Research: [Personalizing Health and Fitness](https://machinelearning.apple.com/research/personalized-heartrate)
- Banister Model: [Modeling Human Performance](https://fellrnr.com/wiki/Modeling_Human_Performance)
- Fitness-Fatigue: [Sports Medicine Open](https://link.springer.com/article/10.1186/s40798-022-00426-x)

### 기술 문서
- [Python Engineer - Apple Health Data](https://www.python-engineer.com/posts/apple-health-data-python/)
- [TRIMP as Training Load](https://ludum.com/blog/data-performance-analytics/trimp-as-a-training-load-score/)
- [Flutter Desktop](https://docs.flutter.dev/platform-integration/desktop)
- [fl_chart](https://pub.dev/packages/fl_chart)

### GitHub 참고
- [apple-health-parser](https://github.com/alxdrcirilo/apple-health-parser)
- [Apple ml-heart-rate-models](https://github.com/apple/ml-heart-rate-models) (참고용)

---

## ✅ 체크리스트

### 개발 전 준비
- [ ] MacBook M1 Python 환경 설정 (3.10+)
- [ ] Flutter Desktop 설치
- [ ] iPhone에서 Health Export 샘플 생성
- [ ] 샘플 ZIP 파싱 테스트
- [ ] 개발 디렉토리 구조 생성

### MVP 완성 조건
- [ ] 3~5개 워크아웃으로 트윈 생성 가능
- [ ] 1~4주 루틴 시뮬레이션 실행
- [ ] HR 그래프 시각화
- [ ] 성장량 수치 표시 (bpm, 피트니스)
- [ ] 과부하 경고 표시
- [ ] 3가지 시나리오 준비

### 공모전 제출
- [ ] 발표 자료 (10-15 슬라이드)
- [ ] 데모 영상 (2-3분)
- [ ] 소스코드 정리 (GitHub)
- [ ] README 작성

---

## 🚀 다음 단계

**바로 시작하기:**
1. Health Export 샘플 받기 (iPhone)
2. XML 파싱 스크립트 작성
3. FastAPI 프로젝트 초기화

**개발 우선순위:**
1. 데이터 파이프라인 (Day 1-2)
2. 백엔드 API (Day 3-7)
3. Flutter UI (Day 8-13)
4. 시연 준비 (Day 14)
