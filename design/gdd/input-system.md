# Input System

> **Status**: In Design
> **Author**: user + game-designer
> **Last Updated**: 2026-04-02
> **Implements Pillar**: "한 손의 몰입" (Effortless Control)

## Overview

Input System은 플레이어의 모든 조작 입력을 추상화하는 Foundation 레이어 시스템이다. 모바일에서는 가상 조이스틱 + 4버튼, PC에서는 키보드+마우스로 동일한 게임플레이 액션(이동, 기본공격, 스킬1/2/3)을 생성한다. 플레이어는 이 시스템을 직접 인식하지 않지만, "한 손의 몰입" 필러의 핵심 기반이다. 입력 장치에 관계없이 동일한 액션 이벤트를 Player Controller에 전달하며, 데드존, 감도, 버튼 배치 등을 커스터마이징할 수 있다.

## Player Fantasy

"조작을 의식하지 않는다." 화면을 터치하는 순간 캐릭터가 내 의지대로 움직이고, 손가락이 떠나면 멈춘다. 버튼을 누르면 즉시 스킬이 발동한다. 조작법을 배우는 시간 제로 — 처음 터치한 순간부터 전투에만 집중할 수 있는 투명한 인터페이스. 소울나이트를 처음 켰을 때 "아 이거 그냥 되네"라는 느낌이 목표다.

## Detailed Design

### Core Rules

1. **입력 추상화 레이어**: 모든 물리 입력(터치, 키보드, 마우스)은 `InputAction`으로 변환된다. 게임 시스템은 물리 입력을 직접 읽지 않고 InputAction만 소비한다.

2. **6가지 InputAction**:
   - `move` — Vector2 (방향 + 크기, 0.0~1.0)
   - `attack` — bool (홀드 시 true 유지 → 자동 연타 콤보)
   - `skill_1`, `skill_2`, `skill_3` — bool (press, 1프레임)
   - `aim` — Vector2 (모바일: 자동조준 시스템이 채움, PC: 마우스 방향)

3. **모바일 입력 매핑**:
   - 왼쪽 고정 가상 조이스틱 → `move`
   - 오른쪽 버튼 4개 → `attack`, `skill_1`, `skill_2`, `skill_3`
   - `aim`은 자동조준 시스템이 채움 (Input System 외부, Player Controller 담당)

4. **PC 입력 매핑**:
   - WASD → `move` (8방향 정규화, 대각선 시 √2 보정)
   - 마우스 좌클릭 → `attack`
   - Q, E, R → `skill_1`, `skill_2`, `skill_3`
   - 마우스 커서 위치 → `aim` (캐릭터 기준 방향 벡터로 변환)

5. **입력 버퍼링**: 스킬 버튼은 60ms 입력 버퍼 적용. 쿨다운 종료 직전 누른 입력도 쿨다운 해제 즉시 발동된다.

6. **동시 입력 처리**:
   - 이동 + 공격/스킬: 동시 허용 (독립 채널)
   - 스킬 + 스킬: 동시 입력 시 먼저 눌린 것만 처리 (FIFO)
   - 공격 + 스킬: 스킬이 우선, 공격 콤보 중단 후 스킬 발동

### States and Transitions

| State | 설명 | 전환 조건 |
|-------|------|-----------|
| **Idle** | 입력 없음 | 모든 입력 해제 시 |
| **Moving** | 조이스틱/WASD 입력 중 | `move.length() > deadzone` |
| **Attacking** | 공격 버튼 홀드 중 | `attack == true` |
| **Moving+Attacking** | 이동+공격 동시 | 두 조건 동시 충족 |
| **Skill Casting** | 스킬 버튼 pressed | `skill_N == true` (1프레임 이벤트) |
| **Disabled** | 입력 차단 (CC, UI, 컷신 등) | 외부 시스템이 `disable_input()` 호출 시 |

- Moving과 Attacking은 독립 채널로 동시 활성 가능
- Disabled 상태에서는 모든 InputAction이 zero/false로 출력
- UI 오픈(인벤토리, 설정 등) 시 게임플레이 입력 자동 Disabled
- Disabled 해제 시 버퍼에 쌓인 입력은 폐기 (해제 후 새 입력만 처리)

### Interactions with Other Systems

| System | 방향 | 인터페이스 |
|--------|------|-----------|
| **Player Controller** | → 출력 | `InputAction` 구조체를 매 프레임 전달. PC가 이동/조준/콤보 로직 처리 |
| **Combat System** | → 출력 (PC 경유) | `attack` 홀드 상태로 자동 콤보 트리거 |
| **Skill System** | → 출력 (PC 경유) | `skill_1/2/3` press 이벤트 전달 |
| **Status Effect System** | ← 입력 | CC(스턴, 빙결) 적용 시 `disable_input()` 요청 수신 |
| **HUD** | ↔ 양방향 | UI 모드 전환 시 입력 차단 요청. 조이스틱/버튼 비주얼은 HUD가 렌더링 |
| **Camera System** | → 출력 | PC에서 `aim` 벡터 전달 (카메라 오프셋/리드 계산용) |
| **Settings** | ← 입력 | 데드존, 감도, 조이스틱 크기/위치, 버튼 크기 설정값 수신 |

## Formulas

### 1. 조이스틱 입력 정규화

```
raw_input = touch_position - joystick_center
if raw_input.length() < DEADZONE:
    move = Vector2.ZERO
else:
    normalized = raw_input.normalized()
    magnitude = clamp((raw_input.length() - DEADZONE) / (JOYSTICK_RADIUS - DEADZONE), 0.0, 1.0)
    move = normalized * magnitude
```

| Variable | 설명 | 기본값 | 범위 |
|----------|------|--------|------|
| `DEADZONE` | 조이스틱 데드존 (반지름 대비 비율) | 0.15 | 0.05 ~ 0.30 |
| `JOYSTICK_RADIUS` | 조이스틱 터치 반지름 (px) | 64 | 48 ~ 96 |
| `magnitude` | 정규화된 이동 크기 | — | 0.0 ~ 1.0 |

### 2. PC aim 방향 계산

```
aim = (mouse_world_position - character_position).normalized()
```

- `mouse_world_position`: 마우스 스크린 좌표를 월드 좌표로 변환한 값
- 출력: 단위 벡터 (length = 1.0)

### 3. 입력 버퍼

```
if skill_pressed AND skill_on_cooldown AND cooldown_remaining <= BUFFER_WINDOW:
    buffer_skill(skill_id, timestamp)

# 쿨다운 해제 시 버퍼 확인
if cooldown_just_ended AND buffer_has(skill_id) AND (now - buffer_timestamp) <= BUFFER_WINDOW:
    execute_skill(skill_id)
```

| Variable | 설명 | 기본값 | 범위 |
|----------|------|--------|------|
| `BUFFER_WINDOW` | 입력 버퍼 윈도우 | 60ms | 30 ~ 120ms |

### 4. WASD 대각선 정규화 (PC)

```
raw = Vector2(
    (is_D_pressed as float) - (is_A_pressed as float),
    (is_S_pressed as float) - (is_W_pressed as float)
)
move = raw.normalized() if raw.length() > 0 else Vector2.ZERO
```

- 대각선 입력 시 속도가 √2배가 되지 않도록 정규화 적용

## Edge Cases

1. **조이스틱 위에서 손가락 이탈**: 터치 종료 시 즉시 `move = Vector2.ZERO`. 조이스틱 영역 밖으로 드래그한 경우는 계속 추적 (터치 ID 기반)
2. **멀티터치 충돌**: 왼쪽 조이스틱 영역과 오른쪽 버튼 영역은 터치 인덱스로 분리. 같은 영역 멀티터치 시 첫 번째 터치만 유효
3. **0fps/극저 프레임**: 입력 버퍼는 실제 시간(ms) 기반 — 프레임 드랍 시에도 버퍼 윈도우는 실시간으로 동작
4. **스턴 중 입력**: Disabled 상태에서 입력은 폐기. 스턴 해제 후 새 입력부터 처리 (버퍼 초기화)
5. **UI 전환 중 홀드 입력**: 인벤토리 열기 순간 attack 홀드 중이었다면 공격 즉시 중단. UI 닫을 때 attack 재입력 필요
6. **PC에서 포커스 이탈**: 게임 윈도우 포커스 잃으면 모든 키 해제 처리. 포커스 복귀 시 현재 눌린 키 상태 재확인
7. **화면 회전 (모바일)**: 가로 모드 고정. 세로 회전 무시
8. **동시 스킬 입력 (같은 프레임)**: 프레임 내 동시 입력 감지 시 낮은 번호 우선 (skill_1 > skill_2 > skill_3)

## Dependencies

### Upstream (이 시스템이 의존하는 시스템)

없음 — Foundation 레이어. 외부 시스템 의존성 없이 독립 동작.

### Downstream (이 시스템에 의존하는 시스템)

| System | 의존 유형 | 설명 |
|--------|-----------|------|
| **Player Controller** | Hard | InputAction을 소비하여 캐릭터 이동, 조준, 공격 처리 |
| **HUD** | Soft | 조이스틱/버튼 비주얼 렌더링, UI 모드 전환 시 입력 차단 요청 |
| **Combat System** | Soft (PC 경유) | attack 상태를 Player Controller를 통해 수신 |
| **Skill System** | Soft (PC 경유) | skill press 이벤트를 Player Controller를 통해 수신 |
| **Camera System** | Soft | PC에서 aim 벡터 수신 (카메라 리드 계산용) |

## Tuning Knobs

| Knob | 기본값 | 안전 범위 | 영향하는 게임플레이 | 극단값 효과 |
|------|--------|-----------|---------------------|-------------|
| `joystick_deadzone` | 0.15 | 0.05 ~ 0.30 | 이동 민감도/정밀도 | 너무 낮으면 조이스틱 드리프트, 너무 높으면 반응 느림 |
| `joystick_radius` | 64px | 48 ~ 96px | 조이스틱 터치 영역 크기 | 너무 작으면 정밀 조작 어려움, 너무 크면 화면 가림 |
| `button_size` | 80px | 60 ~ 120px | 스킬/공격 버튼 터치 영역 | 너무 작으면 미스터치, 너무 크면 화면 가림 |
| `input_buffer_window` | 60ms | 30 ~ 120ms | 스킬 입력 관대함 | 너무 짧으면 입력 씹힘, 너무 길면 의도치 않은 발동 |
| `joystick_opacity` | 0.6 | 0.2 ~ 1.0 | 조이스틱 시각적 존재감 | 낮으면 전투 몰입감 ↑ 조작 가시성 ↓ |

## Visual/Audio Requirements

- 조이스틱: 반투명 원형 베이스 + 이동 방향 표시 썸(thumb)
- 버튼: 아이콘 + 쿨다운 오버레이 (HUD 담당, Input System은 터치 영역만 관리)
- 버튼 탭 시 미세한 진동 피드백 (haptic, 모바일)
- 오디오 없음 — 입력 자체의 소리는 없고, 결과 액션(공격/스킬)의 SFX는 해당 시스템 담당

## UI Requirements

- 조이스틱/버튼의 위치, 크기, 투명도는 설정 화면에서 조절 가능
- 조이스틱은 왼쪽 하단, 버튼은 오른쪽 하단 고정 (위치 커스터마이징은 V2 이후 검토)
- 전투 중 UI 버튼과 게임플레이 터치 영역이 겹치지 않도록 레이아웃 가드 필요

## Acceptance Criteria

1. 모바일에서 조이스틱 터치 시 60fps에서 1프레임 이내(≤16.6ms) 이동 반영
2. 공격 버튼 홀드 시 자동 콤보가 끊기지 않고 반복 실행
3. 스킬 버튼 press 시 쿨다운이 아닌 경우 즉시 발동 (1프레임 이내)
4. 입력 버퍼: 쿨다운 종료 60ms 전 스킬 입력이 쿨다운 해제 즉시 자동 발동
5. 이동 + 공격 동시 조작이 간섭 없이 작동
6. 스턴/빙결 CC 중 모든 게임플레이 입력이 무시되고, 해제 후 정상 복귀
7. UI(인벤토리 등) 오픈 시 게임플레이 입력 완전 차단, 닫으면 복귀
8. PC에서 WASD 대각선 이동 시 속도가 직선과 동일 (정규화 검증)
9. PC에서 마우스 커서 방향으로 캐릭터가 조준
10. 데드존/감도 설정 변경이 실시간 반영 (재시작 불필요)
11. Input System의 프레임당 처리 시간이 0.5ms 이내

## Open Questions

1. **조이스틱 위치 커스터마이징**: MVP에서는 고정 위치, Vertical Slice에서 위치 이동 지원 검토
2. **진동 피드백 세기**: 기기별 haptic API 차이에 따른 표준화 방법 — 프로토타입에서 테스트 필요
3. **PC 키 리바인딩**: MVP에서는 고정 WASD+QER, 추후 키 리바인딩 설정 추가 검토
