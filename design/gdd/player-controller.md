# Player Controller

> **Status**: In Design
> **Author**: user + game-designer
> **Last Updated**: 2026-04-05
> **Implements Pillar**: "한 손의 몰입" (Effortless Control), "한 방의 쾌감" (Loot Dopamine)

## Overview

Player Controller는 Input System의 액션을 소비하여 캐릭터의 이동, 자동조준, 기본공격 콤보, 스킬 발동을 처리하는 Core 레이어 시스템이다. 5개 클래스(Ironclad, Ashcaster, Deadshot, Flickblade, Hivecaller)에 공통되는 조작 로직을 제공하며, 클래스별 차이는 스탯과 스킬 데이터로 분리한다. Input System → Player Controller → Combat/Skill System으로 이어지는 플레이어 액션 파이프라인의 중심 허브.

## Player Fantasy

"내 캐릭터가 내 손끝에 있다." 조이스틱을 밀면 즉시 달리고, 공격 버튼을 누르면 건블레이드가 베기-베기-사격 자동 콤보를 쏟아낸다. 가장 가까운 적을 알아서 조준하니 나는 위치 잡기에만 집중하면 된다. 스킬 버튼 하나로 화면이 폭발한다. 조작 1초 만에 "이 캐릭터 강하다"를 느끼는 순간.

## Detailed Design

### Core Rules

1. **이동**: Input System의 `move` 벡터를 받아 `CharacterBody2D.velocity`에 적용. `move_speed`는 기본값 + 장비/버프 보정으로 결정.

2. **자동조준 (이동 방향 우선)**:
   - 이동 중: 이동 방향 기준 전방 원뿔(±`AIM_CONE_HALF_ANGLE`) 내 가장 가까운 적 우선 타겟
   - 원뿔 내 적 없으면 뷰포트 내 가장 가까운 적으로 폴백
   - 정지 시: 마지막 이동 방향 기준 원뿔 → 뷰포트 내 최근접
   - PC: 마우스 방향(aim 벡터) 사용, 자동조준 비활성
   - 타겟 변경 시 즉시 전환 (스무딩 없음 — 반응성 우선)
   - 타겟이 죽거나 범위 밖으로 나가면 즉시 다음 타겟 재탐색

3. **기본공격 콤보 (베기-베기-사격)**:
   - attack 홀드 시 3단 콤보 자동 반복: Slash1 → Slash2 → Shot → Slash1...
   - 각 단계 간격: `combo_interval` (기본 0.35초, attack_speed로 단축)
   - 콤보 단계별 데미지 배율: Slash1(1.0×), Slash2(1.0×), Shot(1.2×)
   - attack 릴리즈 시 현재 모션 완료 후 중단, 콤보 리셋 타이머 시작
   - `combo_reset_time`(0.8초) 내 재입력 시 다음 단계부터 이어서 진행
   - 리셋 타이머 만료 시 Slash1부터 재시작

4. **스킬 발동**:
   - skill_N press 수신 시 Skill System에 발동 요청
   - 스킬 발동 시 진행 중인 콤보 즉시 중단, 콤보 카운터 리셋
   - 스킬이 쿨다운 중이면 무시 (Input System 버퍼가 처리)

5. **피격 처리**:
   - Damage Calculator로부터 피격 이벤트 수신 → HP 감소
   - 넉백 적용: `knockback_force` × 피격 방향
   - 무적 프레임: `invincibility_duration`(0.2초) 동안 추가 피격 무시

6. **사망**:
   - HP ≤ 0 → Dead 상태 진입
   - 입력 차단, 사망 애니메이션 재생
   - Scene Manager에 결과 화면 전환 요청

### States and Transitions

| State | 설명 | 이동 가능 | 공격 가능 | 전환 조건 |
|-------|------|-----------|-----------|-----------|
| **Idle** | 정지. 입력 대기 | — | — | move == 0, 공격 안 함 |
| **Moving** | 이동 중 | ✓ | — | move.length() > 0 |
| **Attacking** | 콤보 실행 중 | ✓ | — | attack == true |
| **Casting** | 스킬 시전 중 | 스킬별 | ✗ | skill 발동 시 |
| **Hit** | 피격 경직 | ✗ | ✗ | 피격 이벤트 수신 |
| **Dead** | 사망 | ✗ | ✗ | HP ≤ 0 |
| **Disabled** | CC 중 (스턴/빙결) | ✗ | ✗ | Status Effect CC 적용 |

- **Moving + Attacking** 동시 가능 (독립 채널)
- **Casting** 중 이동: 스킬 데이터의 `allow_move` 플래그로 결정
- **Hit** 경직: `hitstun_duration`(0.1초) 후 이전 상태로 복귀
- **Dead** 상태는 불가역 — 결과 화면까지 어떤 상태로도 전환 불가
- **Disabled** 해제 시 Idle로 복귀

### Interactions with Other Systems

| System | 방향 | 인터페이스 |
|--------|------|-----------|
| **Input System** | ← 입력 | `InputAction`(move, attack, skill_1/2/3, aim) 매 프레임 수신 |
| **Camera System** | ← 입력 | 뷰포트 경계 정보(자동조준 범위 계산용), PC aim 벡터 |
| **Combat System** | → 출력 | 콤보 단계 + 현재 타겟 정보 전달 → 실제 히트 판정/데미지 처리 위임 |
| **Skill System** | → 출력 | skill_N 발동 요청 + 현재 타겟 정보 전달 |
| **Damage Calculator** | ← 입력 | 피격 데미지 결과 수신 → HP 감소 적용 |
| **Status Effect System** | ← 입력 | CC(스턴/빙결)/버프/디버프 적용/해제 이벤트 수신 |
| **Equipment Slot** | ← 입력 | 장비 스탯(move_speed, attack_speed, HP 등) 보정치 수신 |
| **Character Growth** | ← 입력 | 레벨업에 따른 기본 스탯 성장치 수신 |
| **HUD** | → 출력 | 현재 HP/최대 HP, 스킬 쿨다운, 활성 버프 목록 → UI 표시 |
| **Scene Manager** | ↔ 양방향 | 방 전환 시 위치/상태 보존, 새 방 스폰 포인트 수신 |

## Formulas

### 1. 이동 속도

```
final_speed = base_speed × (1 + pct_speed_bonus) + flat_speed_bonus
final_speed = max(0, final_speed)
velocity = move_direction × final_speed
```

| Variable | 설명 | 기본값 | 범위 |
|----------|------|--------|------|
| `base_speed` | 기본 이동 속도 (px/s) | 200 | 150 ~ 400 |
| `pct_speed_bonus` | 퍼센트 이동속도 보정 (장비+버프 합산) | 0.0 | -1.0 ~ 1.0 |
| `flat_speed_bonus` | 고정 이동속도 보정 | 0 | -100 ~ 100 |

### 2. 자동조준 타겟 스코어링

```
for each enemy in viewport:
    angle = abs(move_direction.angle_to(direction_to_enemy))
    distance = position.distance_to(enemy.position)

    if angle <= AIM_CONE_HALF_ANGLE:
        score = 1.0 / distance                # 원뿔 내: 거리 기준
    else:
        score = (1.0 / distance) * 0.5         # 원뿔 밖: 50% 패널티

    target = enemy with max(score)
```

| Variable | 설명 | 기본값 | 범위 |
|----------|------|--------|------|
| `AIM_CONE_HALF_ANGLE` | 자동조준 원뿔 반각 | 45° (0.785 rad) | 30° ~ 60° |
| `AUTO_AIM_RANGE` | 자동조준 최대 범위 | 뷰포트 대각선 | — |

### 3. 콤보 타이밍

```
actual_interval = combo_interval / (1 + attack_speed_bonus)
actual_interval = max(MIN_COMBO_INTERVAL, actual_interval)
```

| Variable | 설명 | 기본값 | 범위 |
|----------|------|--------|------|
| `combo_interval` | 기본 콤보 간격 (초) | 0.35 | 0.15 ~ 0.50 |
| `attack_speed_bonus` | 공격속도 보정 (장비+버프) | 0.0 | 0.0 ~ 1.5 (캡) |
| `MIN_COMBO_INTERVAL` | 최소 콤보 간격 (초) | 0.14 | — |
| `combo_reset_time` | 콤보 리셋 유예 시간 (초) | 0.8 | 0.5 ~ 1.5 |

### 4. 콤보 데미지 배율

| Combo Step | Name | Damage Multiplier |
|------------|------|-------------------|
| 1 | Slash1 | 1.0× |
| 2 | Slash2 | 1.0× |
| 3 | Shot | 1.2× |

### 5. 넉백

```
knockback_velocity = hit_direction × KNOCKBACK_FORCE × (1 - knockback_resist)
```

| Variable | 설명 | 기본값 | 범위 |
|----------|------|--------|------|
| `KNOCKBACK_FORCE` | 기본 넉백 강도 (px) | 150 | 50 ~ 300 |
| `knockback_resist` | 넉백 저항 (클래스/장비) | 0.0 | 0.0 ~ 0.8 |

## Edge Cases

1. **적 0마리 시 자동조준**: 타겟 없음 → 마지막 이동 방향으로 공격 (빈 공간에 공격)
2. **타겟이 장애물 뒤**: 시야(LOS) 체크 안 함 — 탑다운 뷰에서 장애물 뒤 적도 타겟 가능
3. **동시 피격 + 스킬**: 피격 경직이 스킬 시전을 캔슬. 경직 해제 후 스킬 재입력 필요
4. **스턴 중 콤보 타이머**: CC 중 콤보 리셋 타이머 정지 → CC 해제 후 이어서 카운트
5. **이동속도 0 이하**: `final_speed = max(0, final_speed)`. 빙결 100% 슬로우 시 이동 불가
6. **사망과 무적 프레임**: 무적 프레임 중에는 데미지 무시 → 사망 불가. 무적 해제 후 피격으로만 사망
7. **방 전환 중 콤보**: 방 전환 시 콤보 리셋, attack 홀드 해제
8. **공격속도 극단값**: attack_speed_bonus 캡(1.5)으로 콤보 간격 최소 0.14초 보장

## Dependencies

### Upstream (이 시스템이 의존하는 시스템)

| System | 의존 유형 | 설명 |
|--------|-----------|------|
| **Input System** | Hard | InputAction 수신 — 이것 없이 동작 불가 |
| **Camera System** | Hard | 뷰포트 정보(자동조준 범위), PC aim 벡터 |
| **Damage Calculator** | Hard | 피격 데미지 결과 수신 |
| **Status Effect System** | Soft | CC/버프/디버프 이벤트 수신 |
| **Equipment Slot** | Soft | 장비 스탯 보정치 수신 |
| **Character Growth** | Soft | 레벨업 스탯 수신 |

### Downstream (이 시스템에 의존하는 시스템)

| System | 의존 유형 | 설명 |
|--------|-----------|------|
| **Combat System** | Hard | 콤보 단계 + 타겟 정보 전달 |
| **Skill System** | Hard | 스킬 발동 요청 + 타겟 전달 |
| **HUD** | Soft | HP, 쿨다운, 버프 상태 표시 |
| **Scene Manager** | Soft | 위치/상태 보존, 스폰 포인트 수신 |

## Tuning Knobs

| Knob | 기본값 | 안전 범위 | 영향하는 게임플레이 | 극단값 효과 |
|------|--------|-----------|---------------------|-------------|
| `base_move_speed` | 200 px/s | 150 ~ 400 | 이동 체감, 회피 용이성 | 낮으면 답답, 높으면 제어 어려움 |
| `combo_interval` | 0.35초 | 0.15 ~ 0.50 | 공격 속도 체감 | 낮으면 정신없음, 높으면 답답 |
| `combo_reset_time` | 0.8초 | 0.5 ~ 1.5 | 콤보 이어가기 관대함 | 낮으면 콤보 끊기기 쉬움, 높으면 항상 이어짐 |
| `aim_cone_half_angle` | 45° | 30° ~ 60° | 자동조준 방향 민감도 | 좁으면 정밀하지만 빗나감, 넓으면 의도와 다른 적 타겟 |
| `hitstun_duration` | 0.1초 | 0.05 ~ 0.3 | 피격 경직감 | 짧으면 피격 무시, 길면 답답 |
| `invincibility_duration` | 0.2초 | 0.1 ~ 0.5 | 무적 프레임 관대함 | 짧으면 연속 피격 지옥, 길면 무적 남용 |
| `knockback_force` | 150 px | 50 ~ 300 | 넉백 거리 | 약하면 피격 안 느껴짐, 강하면 위치 잃음 |
| `shot_damage_multiplier` | 1.2× | 1.0 ~ 1.5 | 3타(사격) 보상감 | 1.0이면 3타 의미 없음, 높으면 3타만 중요 |

## Visual/Audio Requirements

- 이동: 캐릭터 스프라이트 이동 방향 전환 (좌/우 플립)
- 콤보 Slash1/Slash2: 무기 휘두르기 이펙트 (반원형 슬래시 라인)
- 콤보 Shot: 총구 화염 이펙트 + 투사체 스프라이트
- 피격: 캐릭터 스프라이트 깜빡임 (무적 프레임 표시), 히트 이펙트
- 사망: 쓰러짐 애니메이션, 화면 느려짐(히트스톱 0.3초)
- 자동조준 인디케이터: 현재 타겟에 미세한 표시 (원형 마커 등, 폴리시 단계)

## UI Requirements

- HP 바: 캐릭터 상단 또는 HUD 고정 (HUD GDD에서 결정)
- 스킬 쿨다운: 버튼 위 쿨다운 오버레이 (HUD 담당)
- 피격 시 화면 가장자리 적색 비네팅 (체력 낮을 때 강화)

## Acceptance Criteria

1. 이동이 Input 입력 후 1프레임(≤16.6ms) 내 반영
2. 자동조준이 이동 방향 전방 원뿔 내 적을 우선 타겟
3. attack 홀드 시 Slash1→Slash2→Shot 콤보가 끊기지 않고 반복
4. 스킬 입력 시 콤보 즉시 중단되고 스킬 발동
5. 피격 시 넉백 + 무적 프레임(0.2초) 정상 작동
6. HP ≤ 0 시 Dead 상태 진입, 모든 입력 차단
7. 스턴/빙결 CC 중 이동+공격 불가, 해제 후 정상 복귀
8. 이동+공격 동시 시 간섭 없이 둘 다 작동
9. 방 전환 후 플레이어 HP/버프/콤보 상태 정상 보존
10. attack_speed_bonus 최대치에서도 콤보 간격 ≥ 0.14초

## Open Questions

1. **클래스별 콤보 체인 변형**: 모든 클래스가 베기-베기-사격인지, 클래스별 고유 콤보 패턴이 있는지 — Class System GDD에서 결정
2. **대시/회피 스킬**: Player Controller에 대시 메카닉을 기본 제공할지, 스킬로만 제공할지 — Combat System GDD에서 결정
3. **자동조준 타겟 고정**: 현재는 매 프레임 재계산. 특정 적에 타겟 고정(lock-on) 기능이 필요한지 — 프로토타입 테스트 후 결정
