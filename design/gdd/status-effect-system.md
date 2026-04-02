# Status Effect System

> **Status**: In Design
> **Author**: user + game-designer
> **Last Updated**: 2026-03-31
> **Implements Pillar**: "한 방의 쾌감" (Loot Dopamine), "황무지의 색" (Gritty World)

## Overview

전투 중 캐릭터와 몬스터에게 부여되는 일시적 효과를 관리하는 시스템. 화염 DoT, 빙결
감속, 독 지속 데미지, 전기 체인 등 속성 기반 효과와, 버프/디버프(공격력 증가, 방어력
감소 등)를 포함한다. 유니크 아이템의 스킬 변형이 실제 전투에서 발현되는 채널이며,
속성 빌드 다양성의 근간이 된다. 이 시스템이 없으면 데미지는 단순 숫자에 그치고,
유니크 아이템이 "스킬을 바꾸는" 핵심 경험을 전달할 수 없다.

## Player Fantasy

유니크 화염 건블레이드를 장착하고 훨윈드를 시전하면, 적들이 불타오르며 화면이 주황빛으로
물든다. 적이 도망쳐도 화염이 계속 깎아먹고, 다시 때리면 불길이 연장된다. "내 공격이 단순히
맞추는 게 아니라, 적에게 무언가를 남긴다"는 느낌. 반대로 독 늪지에서 독에 걸리면 초록빛이
화면 가장자리를 침범하고, 체력이 서서히 줄어드는 긴장감 — "빨리 벗어나야 한다"는 본능적
반응을 유도한다. 상태 효과는 전투를 단순 타격에서 **전술적 레이어**로 끌어올린다.

## Detailed Design

### Core Rules

**1. 상태 효과 분류**

| 카테고리 | 설명 | 중첩 규칙 |
|----------|------|-----------|
| **DoT (Damage over Time)** | 주기적 데미지 (화염, 독) | 같은 속성: 타이머 리셋 + 높은 데미지 유지. 다른 속성: 독립 중첩 |
| **CC (Crowd Control)** | 이동/행동 제한 (빙결 감속, 전기 스턴) | 같은 CC: 타이머 리셋. 다른 CC: 독립 적용 |
| **Buff** | 아군에게 유리한 효과 (공격력↑, 이동속도↑) | 같은 버프: 높은 쪽 유지 (스택 안 됨). 다른 버프: 독립 적용 |
| **Debuff** | 적에게 불리한 효과 (방어력↓, 공격력↓) | 같은 디버프: 높은 쪽 유지. 다른 디버프: 독립 적용 |

**2. 속성별 상태 효과**

| 속성 | DoT 효과 | CC 효과 | 비고 |
|------|----------|---------|------|
| **Fire** | 화상 (초당 화염 데미지) | 없음 | 순수 데미지 속성. 타격 시 타이머 리셋 |
| **Ice** | 없음 | 감속 (이동속도 -30~50%) | 일정 스택 누적 시 완전 빙결 (1-2초 행동 불가) |
| **Poison** | 중독 (초당 독 데미지) | 없음 | 화염보다 틱 데미지 낮지만 지속시간 길음 |
| **Electric** | 없음 | 짧은 스턴 (0.3-0.5초) + 주변 적 체인 히트 | 다수 적에게 효과적 |

**3. 효과 적용 흐름**

1. 트리거: 공격/스킬이 속성 데미지를 포함하여 적중
2. 확률 체크: 효과 적용 확률 판정 (100% 고정 또는 확률 기반, 소스에 따라)
3. 기존 효과 체크: 같은 효과가 이미 있는지 확인
   - 있으면: 중첩 규칙 적용 (타이머 리셋, 데미지 비교)
   - 없으면: 새 효과 인스턴스 생성
4. 틱 처리: tick_interval마다 Damage Calculator 호출 (DoT) 또는 CC 효과 유지
5. 만료: duration 도달 시 효과 제거, 비주얼/오디오 클린업

**4. 효과 인스턴스 데이터**

각 활성 효과는 다음 데이터를 보유:

```
StatusEffectInstance:
  effect_type: String        # "fire_dot", "ice_slow", "poison_dot", "electric_stun"
  category: Enum             # DOT, CC, BUFF, DEBUFF
  element: Enum              # FIRE, ICE, POISON, ELECTRIC, NONE
  source: Entity             # 효과를 건 주체 (플레이어/몬스터)
  target: Entity             # 효과를 받는 대상
  dot_base: int              # DoT 틱당 기본 데미지 (DoT만)
  elemental_pct_increase: float  # 속성 증가% (DoT만)
  cc_value: float            # CC 수치 (감속률, 스턴 시간 등)
  duration: float            # 총 지속 시간 (초)
  remaining_time: float      # 남은 시간
  tick_interval: float       # 틱 간격 (DoT만)
  tick_timer: float          # 다음 틱까지 남은 시간
```

### States and Transitions

각 효과 인스턴스의 생명주기:

| State | Entry Condition | Exit Condition | Behavior |
|-------|----------------|----------------|----------|
| **Inactive** | 초기 상태 / 만료 후 | 효과 적용 트리거 | 아무 동작 없음 |
| **Active** | 효과 적용 성공 | remaining_time ≤ 0 또는 명시적 제거 | 매 틱마다 효과 발동 (DoT: 데미지, CC: 상태 유지) |
| **Refreshed** | 같은 효과 재적용 | 즉시 Active로 복귀 | remaining_time 리셋, dot_base는 max(기존, 새것) |
| **Expired** | remaining_time ≤ 0 | 즉시 Inactive로 | 비주얼/오디오 클린업, 인스턴스 풀로 반환 |

**빙결 특수 상태** (Ice 속성):

| State | Entry Condition | Exit Condition | Behavior |
|-------|----------------|----------------|----------|
| **Chilled** | Ice 공격 적중 | freeze_stacks ≥ freeze_threshold 또는 만료 | 이동속도 감소, 빙결 스택 누적 |
| **Frozen** | freeze_stacks ≥ threshold | frozen_duration 만료 | 완전 행동 불가 (1-2초), 스택 리셋 |

### Interactions with Other Systems

| 시스템 | 방향 | 인터페이스 |
|--------|------|-----------|
| **Damage Calculator** | this → 호출 | DoT 틱마다 `calculate_damage()` 호출. dot_base, elemental_pct_increase 전달, 방어력 0으로 전달 (DoT 방어 무시 규칙) |
| **Combat System** | ← 트리거 | 공격 적중 시 `apply_status_effect(target, effect_data)` 호출. 속성 데미지가 포함된 공격만 효과 트리거 |
| **Skill System** | ← 트리거 | 스킬 적중 시 효과 적용. 스킬 데이터에 적용할 효과 타입/확률 포함 |
| **Skill Mutation System** | ← 데이터 제공 | 유니크 아이템이 스킬에 속성을 추가하면, 해당 스킬의 effect_data에 상태 효과 정보 삽입 |
| **Monster/Enemy AI** | 양방향 | 몬스터도 플레이어에게 상태 효과 적용 가능. 몬스터 공격 데이터에 effect_data 포함 |
| **Combat Feedback** | → 출력 | 효과 적용/틱/만료 시 이벤트 발생 → 파티클, 색상 변화, 데미지 숫자 표시 |
| **HUD** | → 출력 | 플레이어에게 적용된 활성 효과 아이콘 + 남은 시간 표시 |

## Formulas

### DoT 틱 데미지 (Damage Calculator GDD 참조)

```
dot_tick_damage = floor(dot_base × (1 + elemental_pct_increase))
```

방어력 무시 (Damage Calculator 설계 결정).

### 화염 (Fire) — 총 DoT 데미지

```
total_fire_damage = dot_tick_damage × (duration / tick_interval)
```

| Variable | Type | Default | Range | Description |
|----------|------|---------|-------|-------------|
| dot_base | int | 5 | 3-50 | 틱당 기본 화염 데미지 |
| tick_interval | float | 0.5 | 0.5-1.0 | 초 단위 틱 간격 |
| duration | float | 3.0 | 2.0-5.0 | 지속 시간 (초) |

**예시**: dot_base=5, pct_increase=0, duration=3.0, tick_interval=0.5
→ 틱당 5 × 6틱 = **총 30 데미지** (초반 기준, 기본 공격 5-15 대비 상당한 추가 가치)

### 독 (Poison) — 낮은 틱, 긴 지속

| Variable | Type | Default | Range | Description |
|----------|------|---------|-------|-------------|
| dot_base | int | 3 | 2-30 | 틱당 기본 독 데미지 (화염보다 낮음) |
| tick_interval | float | 1.0 | 0.5-1.0 | 더 느린 틱 |
| duration | float | 5.0 | 3.0-8.0 | 화염보다 긴 지속 |

**예시**: dot_base=3, duration=5.0, tick_interval=1.0
→ 틱당 3 × 5틱 = **총 15 데미지** (화염보다 낮지만 지속이 길어 도주하는 적에 유효)

### 빙결 (Ice) — 감속 + 스택

```
slow_amount = base_slow × (1 + ice_pct_bonus)
```

| Variable | Type | Default | Range | Description |
|----------|------|---------|-------|-------------|
| base_slow | float | 0.3 | 0.2-0.5 | 기본 감속률 (30%) |
| ice_pct_bonus | float | 0.0 | 0.0-0.5 | 아이템/스킬 빙결 강화 |
| freeze_threshold | int | 5 | 3-8 | 완전 빙결까지 필요한 스택 수 |
| freeze_duration | float | 1.5 | 1.0-2.0 | 완전 빙결 지속 시간 |
| chill_duration | float | 3.0 | 2.0-5.0 | 감속 지속 시간 (스택당 리셋) |

### 전기 (Electric) — 스턴 + 체인

```
chain_damage = floor(original_damage × chain_damage_ratio)
```

| Variable | Type | Default | Range | Description |
|----------|------|---------|-------|-------------|
| stun_duration | float | 0.3 | 0.2-0.5 | 짧은 스턴 시간 |
| chain_range | float | 64.0 | 48-96 | 체인 히트 범위 (픽셀) |
| chain_count | int | 2 | 1-4 | 최대 체인 대상 수 |
| chain_damage_ratio | float | 0.5 | 0.3-0.7 | 원본 대비 체인 데미지 비율 |

## Edge Cases

| Scenario | Expected Behavior | Rationale |
|----------|------------------|-----------|
| 같은 속성 DoT 재적용, 새 dot_base가 더 낮음 | 타이머만 리셋, 기존 높은 dot_base 유지 | 강한 효과가 약한 것으로 덮어쓰여지면 안 됨 |
| 같은 속성 DoT 재적용, 새 dot_base가 더 높음 | 타이머 리셋 + dot_base 교체 | 더 강한 장비의 가치 보장 |
| 4종 속성 DoT/CC 동시 적용 | 모두 독립 유지 (최대 4개 동시) | 멀티 속성 빌드 보상 |
| 빙결 스택 중 빙결 만료 | 스택 리셋, 감속 해제 | 지속적으로 때려야 빙결 가능 |
| Frozen 상태에서 추가 Ice 공격 | 무시 (이미 빙결 중) | 빙결 무한 연장 방지 |
| 전기 체인이 이미 전기 상태인 적에 적중 | 스턴 타이머 리셋, 체인은 발생하지 않음 | 무한 체인 루프 방지 |
| 보스에게 CC 적용 | CC 효과 시간 50% 감소 (보스 저항) | 보스가 너무 쉽게 무력화되면 안 됨 |
| 효과 100개 이상 동시 활성 (화면 내 다수 적) | 오래된 효과부터 만료, 최대 동시 인스턴스 캡 적용 | 성능 보호 |
| 플레이어 사망 시 활성 효과 | 모든 효과 즉시 제거 | 마을 복귀 시 클린 상태 |
| 던전 방 전환 시 | 플레이어 효과 유지, 몬스터 효과 제거 | 방 전환 후 이전 방 몬스터는 없으므로 |

## Dependencies

| System | Direction | Nature |
|--------|-----------|--------|
| **Damage Calculator** | this depends on | Hard — DoT 틱 데미지 계산에 필수 |
| **Combat System** | depends on this | Hard — 속성 공격 시 효과 적용 트리거 |
| **Skill System** | depends on this | Hard — 스킬의 속성 효과 발동 |
| **Skill Mutation System** | depends on this | Hard — 유니크 아이템의 스킬 변형이 이 시스템으로 효과 추가 |
| **Monster/Enemy AI** | depends on this | Hard — 몬스터 속성 공격의 효과 적용 |
| **Combat Feedback** | depends on this | Soft — 효과 비주얼/사운드 표시 |
| **HUD** | depends on this | Soft — 활성 효과 아이콘 표시 |

## Tuning Knobs

| Parameter | Current Value | Safe Range | Effect of Increase | Effect of Decrease |
|-----------|--------------|------------|-------------------|-------------------|
| fire_dot_base | 5 | 3-50 | 화염 빌드 데미지↑ | 화염 빌드 약화 |
| fire_duration | 3.0 | 2.0-5.0 | 화염 총 데미지↑, 시각 효과 길어짐 | 짧은 버스트, 리프레시 중요도↑ |
| poison_dot_base | 3 | 2-30 | 독 빌드 데미지↑ | 독 빌드 약화 |
| poison_duration | 5.0 | 3.0-8.0 | 도주 적에 더 유효 | 직접 전투 중 가치 감소 |
| ice_base_slow | 0.3 | 0.2-0.5 | 빙결 빌드 CC 강화, 카이팅 쉬워짐 | 빙결 빌드 CC 약화 |
| freeze_threshold | 5 | 3-8 | 완전 빙결 어려워짐, 보스전 밸런스 | 빙결 빌드 너무 강해짐 |
| freeze_duration | 1.5 | 1.0-2.0 | 빙결 중 무료 공격 시간↑ | 빙결의 가치 감소 |
| electric_stun_duration | 0.3 | 0.2-0.5 | 스턴 빌드 강화 | 스턴 느낌 약화 |
| electric_chain_count | 2 | 1-4 | 다수 적에 강해짐, AoE 가치 | 단일 대상에 집중 |
| boss_cc_resistance | 0.5 | 0.3-0.7 | 보스에 CC 거의 무효 | 보스 CC로 쉽게 무력화 |
| max_active_effects | 64 | 32-128 | 메모리 사용↑, 대규모 전투 지원 | 성능 보호, 효과 자동 만료 |

## Visual/Audio Requirements

| Event | Visual Feedback | Audio Feedback | Priority |
|-------|----------------|---------------|----------|
| 화상 적용 | 대상에 화염 파티클 루프 | 불꽃 소리 | MVP |
| 화상 틱 | 작은 주황 숫자 팝업 | 약한 화염 틱 소리 | MVP |
| 화상 만료 | 파티클 페이드아웃 | — | MVP |
| 중독 적용 | 대상에 녹색 독기 파티클 | 독 효과음 | Vertical Slice |
| 감속 (Chilled) | 대상 스프라이트 하늘색 틴트 | 빙결 시작 효과음 | Vertical Slice |
| 완전 빙결 (Frozen) | 얼음 오버레이 + 움직임 정지 | 얼음 깨지는 소리 (해제 시) | Vertical Slice |
| 전기 스턴 | 전기 스파크 파티클 | 전기 효과음 | Vertical Slice |
| 전기 체인 | 대상 간 전기 선 이펙트 | 체인 효과음 | Vertical Slice |
| 플레이어 피해 효과 | 화면 가장자리 속성색 오버레이 | — | Vertical Slice |

## UI Requirements

| Information | Display Location | Update Frequency | Condition |
|-------------|-----------------|-----------------|-----------|
| 활성 효과 아이콘 | HUD 캐릭터 상태 영역 | 효과 적용/만료 시 | 플레이어에 효과 있을 때 |
| 남은 시간 바 | 효과 아이콘 아래 | 매 프레임 | 활성 효과 있을 때 |
| 적 상태 효과 | 적 HP바 아래 작은 아이콘 | 효과 적용/만료 시 | 적에 효과 있을 때 |

## Acceptance Criteria

- [ ] 4종 속성(Fire/Ice/Poison/Electric) 효과가 각각 정상 적용된다
- [ ] 같은 속성 DoT 재적용 시 타이머 리셋 + 높은 데미지 유지
- [ ] 다른 속성 효과는 독립적으로 중첩된다
- [ ] DoT 틱 데미지가 Damage Calculator를 거쳐 정확히 계산된다
- [ ] DoT에 방어력이 적용되지 않는다
- [ ] Ice 감속률이 정확히 적용되고, 스택 누적 시 Frozen으로 전이한다
- [ ] Frozen 상태에서 추가 Ice 공격이 빙결을 연장하지 않는다
- [ ] Electric 체인이 range 내 최대 chain_count 대상에게 적중한다
- [ ] 전기 체인이 무한 루프하지 않는다
- [ ] 보스에게 CC 시간이 50% 감소 적용된다
- [ ] 플레이어 사망 시 모든 효과가 즉시 제거된다
- [ ] 방 전환 시 플레이어 효과는 유지, 몬스터 효과는 제거된다
- [ ] 동시 활성 효과가 max_active_effects를 초과하지 않는다
- [ ] Performance: 64개 동시 효과 틱 처리가 1ms 이내 완료
- [ ] 모든 수치가 외부 데이터 파일에서 로드되며 하드코딩 없음

## Open Questions

| Question | Owner | Deadline | Resolution |
|----------|-------|----------|-----------|
| 속성 저항(elemental resistance) 스탯을 도입할지? | game-designer | Item System 설계 시 | Damage Calculator Open Question에서도 언급 |
| 속성 간 시너지 효과를 추가할지? (예: 빙결+전기=폭발) | game-designer | Skill Mutation System 설계 시 | — |
| 환경 상태 효과(독 늪 = 자동 중독 등)의 구체적 수치 | level-designer | Dungeon Generator 설계 시 | — |
