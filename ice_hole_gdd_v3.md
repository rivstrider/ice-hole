# ICE HOLE — 게임 기획서 v3.0

---

## 1. 게임 개요

| 항목 | 내용 |
|------|------|
| 장르 | 캐주얼 미니게임 |
| 플랫폼 | 웹 (HTML5 Canvas) |
| 제한시간 | 90초 |
| 목표 | 제한시간 내 최대 포인트 획득 |
| 조작 | 마우스 왼쪽 버튼 (클릭=훅셋 / 홀드=릴링) |
| 기준 해상도 | 1920×1080 (세로 기준 스케일, 모바일 좌우 크롭) |

---

## 2. 게임 상태 머신

```
IDLE → CASTING → WAITING → BITE → FIGHTING → LANDING
                                 ↘            ↘
                                  FAIL(훅아웃)  FAIL(라인터짐)
                                       ↓
                                    CASTING (자동 재캐스팅)
```

| 상태 | 설명 |
|------|------|
| `IDLE` | 초기 대기 (게임 시작 전) |
| `CASTING` | 자동 캐스팅, lineLength 0→depth 증가 |
| `WAITING` | 입질 대기 중 |
| `BITE` | 입질 패턴 실행 중 |
| `FIGHTING` | 피쉬온, 릴링으로 라인 회수 |
| `LANDING` | 랜딩 성공 처리 |
| `FAIL` | 라인 터짐 / 훅아웃 실패 처리 |

---

## 3. 화면 구성 (좌표 기준)

| 요소 | 값 | 설명 |
|------|-----|------|
| `SCALE` | `H / 1080` | 세로 기준 스케일 계수 |
| `holeX` | `W / 2` | 얼음구멍 중심 X (화면 가로 중앙) |
| `holeY` | `H - 400 * SCALE` | 얼음구멍 중심 Y (화면 바닥에서 400px 위) |
| `holeRx` | `150 * SCALE` | 구멍 가로 반지름 → 지름 300px |
| `holeRy` | `80 * SCALE` | 구멍 세로 반지름 → 지름 160px |

---

## 4. 핵심 변수 정의

### 4-1. 게임 진행

| 변수 | 초기값 | 설명 |
|------|--------|------|
| `timeLeft` | 90 | 남은 시간 (초) |
| `totalScore` | 0 | 누적 포인트 |
| `catchCount` | 0 | 총 랜딩 성공 횟수 |

### 4-2. 라인

| 변수 | 범위 | 설명 |
|------|------|------|
| `depth` | 300 (고정) | 구멍의 수심 (lineLength 최대값) |
| `lineLength` | 0 ~ depth | 남은 라인 길이. 0이 되면 랜딩 성공 |
| `lineTension` | 0 ~ 100+ | 라인 부하. 100 초과 0.5초 지속 시 라인 터짐 |
| `reelRetrieve` | 250 (고정) | 릴링 시 기본 라인 회수 속도 (px/초) |

### 4-3. 찌

| 변수 | 범위 | 설명 |
|------|------|------|
| `floatDive` | 0 ~ 100+ | 영점(holeY)에서 아래로 내려간 절대 픽셀 거리 |
| `floatY` | - | 찌의 실제 Y좌표 = `holeY + floatDive` |
| `fr` | 20 * SCALE | 찌 반지름 (지름 40px @ 1080p) |

> 찌는 영점 위로 이동 불가. 아래 방향으로만 이동.

### 4-4. 입질

| 변수 | 설명 |
|------|------|
| `biteRate` | 1.5 — 초당 입질 발생 빈도 |
| `biteTimer` | 다음 입질까지 남은 시간 (초) |
| `biteCooldown` | 입질 종료 후 쿨다운 시간 (초) |
| `currentBiteType` | 현재 입질 패턴 (`NIBBLE` / `FAKE` / `SUDDEN_DIVE`) |
| `biteProgress` | 현재 입질 패턴의 경과 시간 (초) |
| `biteRising` | `true` = 찌가 떠오르는 중 (이 동안 훅셋 불가) |

### 4-5. 물고기

| 변수 | 범위 | 설명 |
|------|------|------|
| `fishWeight` | 0.5 ~ 4.0 kg | 물고기 무게. 랜덤 생성 |
| `fishPower` | = fishWeight | 라인텐션에 가하는 기본 부하 |
| `fishActivity` | 0.5 (고정) | fishPower 실시간 변동폭 (±50%) |
| `fishBurst` | 1.0 (고정) | 버스트 시 fishPower 배수 |
| `fishBurstRate` | 0.5 (고정) | 버스트 발생 빈도 (초당 0.5회) |
| `currentPower` | - | 매 프레임 변동되는 실시간 fishPower |
| `effectivePower` | - | 버스트 여부 반영된 최종 부하값 |

### 4-6. 파이팅 안전장치

| 변수 | 설명 |
|------|------|
| `peakTension` | 파이팅 중 텐션 최고값 추적 |
| `lowTensionTimer` | 텐션 저하 지속 시간 (훅아웃 판정용) |
| `highTensionTimer` | 텐션 초과 지속 시간 (라인터짐 판정용) |

### 4-7. 콤보

| 변수 | 설명 |
|------|------|
| `combo` | 현재 연속 훅셋 성공 횟수 |
| `comboTimer` | 콤보 유지 타이머 (12초, 이 안에 다음 훅셋 못하면 리셋) |

---

## 5. 수식 정의

### 5-1. 실시간 fishPower 변동 (매 프레임)

```
currentPower = fishPower × (1 + random(-fishActivity, +fishActivity))
             = fishWeight × (0.5 ~ 1.5 범위에서 랜덤)
```

### 5-2. 버스트 적용

```
effectivePower = isBursting
  ? currentPower × 2 × fishBurst   // 버스트 시 2배
  : currentPower
```

### 5-3. 릴링 중 텐션 / 라인 변화

```
lineTension += effectivePower × 20 × dt
effectiveRetrieve = reelRetrieve / (1 + fishWeight × 0.3)
lineLength -= effectiveRetrieve × dt
```

**effectiveRetrieve 예시 (reelRetrieve=250, k=0.3)**

| fishWeight | effectiveRetrieve |
|-----------|-------------------|
| 0.5kg | 182 px/s |
| 2.0kg | 143 px/s |
| 4.0kg | 108 px/s |

### 5-4. 릴링 해제 시 텐션 / 라인 변화

```
lineTension -= effectivePower × 20 × dt
lineLength  += effectivePower × 6 × dt
```

### 5-5. 캐스팅 속도

```
castProgress += dt × (reelRetrieve / depth) × 2
lineLength = castProgress × depth
```

---

## 6. 판정 조건

### 6-1. 훅셋 성공
```
state == BITE
&& floatDive >= 20      // 찌가 영점에서 20px 이상 내려간 상태
&& !biteRising          // 떠오르는 중이 아닐 것
```

### 6-2. 라인 터짐
```
lineTension >= 100 → highTensionTimer += dt
highTensionTimer >= 0.5초 → LINE BREAK
```

### 6-3. 훅아웃 (바늘 빠짐)
```
!mouseDown
&& lineTension < max(peakTension × 0.9, 15)
→ lowTensionTimer += dt
lowTensionTimer >= 1.5초 → HOOK OUT
```
> 릴링 중에는 카운트 안 됨. 릴을 놓고 있을 때만 감지.

### 6-4. 랜딩 성공
```
lineLength <= 0 → LANDING
```

---

## 7. 포인트 & 콤보

### 포인트 계산
```
basePt = round(fishWeight × 10)
pt = basePt × combo
totalScore += pt
```

### 콤보 시스템

| 조건 | 동작 |
|------|------|
| 훅셋 성공 | `combo++`, `comboTimer = 12초` 리셋 |
| 12초 내 다음 훅셋 성공 | 콤보 유지 및 증가 |
| 12초 경과 | `combo = 0` 리셋 |
| MISS (훅셋 실패) | `combo = 0` 즉시 리셋 |
| 라인 터짐 / 훅아웃 | `combo = 0` 즉시 리셋 |

**콤보 배율 예시**

| 콤보 | 배율 | 1.0kg 물고기 점수 |
|------|------|-------------------|
| x1 | 1배 | 10pt |
| x2 | 2배 | 20pt |
| x3 | 3배 | 30pt |
| xN | N배 | N × basePt |

---

## 8. 입질 패턴

### 패턴 등장 빈도
| 패턴 | 빈도 |
|------|------|
| NIBBLE | 25% |
| FAKE | 25% |
| SUDDEN_DIVE | 50% |

---

### NIBBLE (깔짝)
- **구조**: sin 반주기로 내려갔다 올라오는 다이브 2~5회 반복
- **각 다이브 깊이**: 3~10px
- **각 다이브 속도**: 100px당 0.15~0.2초 (절대속도 기준)
- **각 다이브 간격**: 0.15~0.5초
- **훅셋**: 불가 (최대 깊이 10px < 훅셋 기준 20px)
- **종료 후 쿨다운**: 0초

```
duration = depth × diveSpeed / 100 × 2  // sin 반주기 시간
floatDive = sin(t × π) × depth          // t: 0→1
```

---

### FAKE (가짜 입질)
- **구조**: 단발 다이브 → 떠오르기
- **최대 깊이**: 3~16px
- **잠기는 속도**: 100px당 0.25~0.8초 (절대속도 기준)
- **홀드타임**: 없음
- **떠오르는 속도**: 150px/초 (최소 0.5초 보장)
- **훅셋**: 불가 (최대 깊이 16px < 훅셋 기준 20px)
- **종료 후 쿨다운**: 0.5~1.0초

```
diveTime  = maxDepth × diveSpeed / 100
riseTime  = max(maxDepth / 150, 0.5)

// 다이브: easeOut
floatDive = (1 - (1-t)²) × maxDepth

// 라이즈: easeIn (훅셋 불가 구간)
floatDive = maxDepth × (1 - t²)
```

---

### SUDDEN_DIVE (갑자기 잠김)
- **구조**: 단발 다이브 → 홀드 → 떠오르기
- **최대 깊이**: 20~100px (100px 이상이면 찌 완전 소멸)
- **잠기는 속도**: 100px당 0.25~0.8초 (절대속도 기준)
- **홀드타임**: 0.2초 (고정)
- **떠오르는 속도**: 150px/초 (최소 0.5초 보장)
- **훅셋 가능 구간**: 다이브 중 + 홀드 중 (`floatDive >= 20 && !biteRising`)
- **훅셋 불가 구간**: 라이즈 중 (`biteRising = true`)
- **종료 후 쿨다운**: 0.5~1.0초

```
diveTime  = maxDepth × diveSpeed / 100
holdTime  = 0.2
riseTime  = max(maxDepth / 150, 0.5)

// 다이브: easeOut
floatDive = (1 - (1-t)²) × maxDepth

// 홀드: floatDive = maxDepth (훅셋 가능)

// 라이즈: easeIn (훅셋 불가)
floatDive = maxDepth × (1 - t²)
```

---

## 9. 버스트 시스템

| 항목 | 값 |
|------|-----|
| 발생 빈도 | `fishBurstRate = 0.5` → 평균 2초에 1회 |
| 지속 시간 | 0.3~0.5초 랜덤 |
| 텐션 부하 | `effectivePower × 2` (평상시의 2배) |
| 시각 효과 | 텐션 게이지 붉은 플래시 + 물방울 파티클 10개 |

```
burstTimer = (1 / fishBurstRate) × (0.8 ~ 1.2)  // 랜덤 주기
tensionFlash → 0 (dt × 4 속도로 감소)
```

---

## 10. 파문 시스템

| 생성 조건 | 설명 |
|-----------|------|
| 캐스팅 시작 시 | castProgress = 0 순간 1회 |
| 찌 다이브 중 | `floatDive > lastFloatDive + 2` 마다 1회 |

```
// 애니메이션
t = 1 - ripple.life          // 0=시작, 1=끝
rx = fr + (holeRx - fr) × t  // 찌 크기 → 구멍 크기로 확장
ry = rx × (holeRy / holeRx)
alpha = life × 0.25
life -= dt × 0.7
```

---

## 11. 콤보 메시지 애니메이션

| 페이즈 | 지속 | 동작 |
|--------|------|------|
| `enter` | 0.1초 | 아래에서 40px 위로 빠르게 등장, alpha 0→1 |
| `hold` | 12초 | 서서히 30px 위로 올라가며 alpha 1→0.2 |
| `exit` | 0.15초 | 빠르게 사라짐, alpha →0 |

- **등장 조건**: 훅셋 성공 시 (combo >= 1 이지만 화면 표시는 x1부터)
- **해제 조건**: MISS / 라인터짐 / 훅아웃 / 콤보 타임아웃

---

## 12. HUD 구성

| 위치 | 표시 내용 |
|------|-----------|
| 좌상단 | TIME (남은 시간) / SCORE (pt) / BEST |
| 우측 | TENSION 세로 게이지 (파이팅 중만 표시) |
| 하단 | LINE LENGTH 가로 슬라이더 + 바늘/물고기 아이콘 |
| 구멍 위 | showMsg 일시 메시지 (FISH ON! / MISS! / HOOK OUT! / LINE BREAK! / +Npt) |
| 화면 중앙 | 콤보 메시지 (x2 COMBO! 이상) |

---

## 13. 조작

| 입력 | 상태 | 동작 |
|------|------|------|
| 마우스 좌클릭 | WAITING / BITE | 훅셋 시도 |
| 마우스 좌클릭 홀드 | FIGHTING | 릴링 (누르는 동안 유지) |

---

## 14. 실패 유형 정리

| 유형 | 조건 | 메시지 | 콤보 |
|------|------|--------|------|
| 훅셋 미스 | floatDive < 20 또는 biteRising | MISS! | 즉시 리셋 |
| 라인 터짐 | lineTension ≥ 100을 0.5초 유지 | LINE BREAK! | 즉시 리셋 |
| 훅아웃 | 릴 해제 + 텐션 저하 1.5초 유지 | HOOK OUT! | 즉시 리셋 |

---

*기획서 버전 3.0 / 프로토타입 기준*
