# Damage Calculator

> **Status**: Designed
> **Author**: user + game-designer
> **Last Updated**: 2026-03-30
> **Implements Pillar**: "한 방의 쾌감" (Loot Dopamine), "20분의 밀도" (Dense Runs)

## Overview

IRONCLAW의 모든 데미지 연산을 담당하는 중앙 계산 모듈. 공격자의 공격력, 무기 데미지,
속성 보너스, 크리티컬, 방어력 감소를 거쳐 최종 데미지 숫자를 산출한다. 플레이어→몬스터,
몬스터→플레이어 양방향 모두 이 시스템을 거치며, Combat System, Skill System, Status
Effect System이 이 모듈의 출력을 소비한다. 이 시스템이 없으면 전투에서 아무 숫자도
나오지 않는다.

## Player Fantasy

플레이어는 데미지 계산기를 직접 인식하지 않지만, 이 시스템의 출력이 곧 "성장의 쾌감"이다.
새 유니크 장비를 끼고 던전에 들어갔을 때, 데미지 숫자가 눈에 띄게 커지는 순간 — 그 수치
점프가 "한 방의 쾌감" 필러를 전달하는 핵심 채널이다. 반대로, 상위 던전에서 적의 방어력이
높아 데미지가 줄어드는 순간은 "더 강한 장비가 필요하다"는 파밍 동기를 만든다.
이 시스템은 성장감과 도전감의 균형추 역할을 한다.

## Detailed Design

### Core Rules

**1. 데미지 계산 파이프라인**

모든 데미지는 다음 순서를 거쳐 최종 값이 산출된다:

```
Raw Damage → Flat Bonus → % Increase → Critical → Elemental → Defense Reduction → Final Damage
```

1. **Base Damage 산출**: 무기 기본 데미지 + 공격자 스탯 기반 플랫 보너스
2. **% Increase 적용**: 아이템/버프의 데미지 증가% 합산 후 곱연산
3. **Critical 판정**: 크리티컬 확률 체크 → 성공 시 크리티컬 배율 곱연산
4. **Elemental 배율**: 속성 데미지 보너스 적용 (화염/빙결/독/전기)
5. **Defense 감소**: 피격자의 방어력만큼 고정값 차감
6. **최종 클램핑**: 최소 1 데미지 보장 (0이나 음수 방지)

**2. 데미지 타입**

| 타입 | 설명 | 예시 |
|------|------|------|
| Physical | 기본 물리 데미지, 방어력으로 감소 | 건블레이드 베기, 사격 |
| Fire | 화염 속성, 추가 DoT 가능 | 유니크 화염 건블레이드 |
| Ice | 빙결 속성, 이동속도 감소 가능 | 빙결 스킬 |
| Poison | 독 속성, 지속 데미지 | 늪지 몬스터 공격 |
| Electric | 전기 속성, 체인 히트 가능 | 전기 스킬 |

모든 속성 데미지도 동일한 파이프라인을 거친다. 물리 데미지와 속성 데미지는
**별도 계산 후 합산**된다.

**3. 양방향 적용**

- 플레이어 → 몬스터: 플레이어의 공격력/무기/아이템 옵션 → 몬스터 방어력
- 몬스터 → 플레이어: 몬스터 공격력 → 플레이어 방어력/장비 방어력
- 동일한 파이프라인 사용, 입력값만 다름

**4. 데미지 소스 구분**

| 소스 | 설명 |
|------|------|
| Basic Attack | 기본 공격 (자동 콤보) |
| Skill | 액티브 스킬 (별도 배율 적용) |
| DoT (Damage over Time) | 상태 효과 틱 데미지 (Status Effect System에서 호출) |
| Environmental | 환경 데미지 (독 늪, 함정 등) |

### States and Transitions

데미지 계산기는 상태를 갖지 않는 순수 계산 모듈이다. 매 호출 시 입력 파라미터를 받아
최종 데미지를 반환하며, 호출 간에 내부 상태를 보존하지 않는다. 상태 관리는 호출하는
시스템(Combat System, Status Effect System)의 책임이다.

### Interactions with Other Systems

| 시스템 | 방향 | 인터페이스 |
|--------|------|-----------|
| **Combat System** | → 호출자 | 기본 공격/콤보 타격 시 `calculate_damage(attacker, defender, source_type)` 호출. 최종 데미지 int 반환. |
| **Skill System** | → 호출자 | 스킬 발동 시 호출. 스킬 배율(`skill_multiplier`)을 추가 파라미터로 전달. |
| **Status Effect System** | → 호출자 | DoT 틱마다 호출. 이미 계산된 속성 데미지를 기반으로 틱당 데미지 산출. |
| **Monster/Enemy AI** | → 호출자 | 몬스터 공격 시 호출. 몬스터 스탯이 attacker로 입력. |
| **Item System** | ← 데이터 제공 | 장비의 공격력, 방어력, % 증가, 속성 보너스 등을 attacker/defender 스탯에 반영. 데미지 계산기가 직접 아이템을 읽지 않음 — 호출자가 합산된 스탯을 전달. |
| **Character Growth** | ← 데이터 제공 | 레벨 기반 기본 스탯을 제공. 역시 호출자가 합산해서 전달. |
| **Combat Feedback** | ← 출력 소비 | 최종 데미지 + 크리티컬 여부 + 속성 타입을 받아 데미지 숫자 팝업/이펙트 표시. |

**핵심 원칙**: 데미지 계산기는 스탯을 직접 조회하지 않는다. 호출자가 모든 관련 스탯을
합산하여 전달하고, 계산기는 받은 값으로만 연산한다. 이렇게 하면 시스템 간 결합도가
낮아지고 테스트가 쉬워진다.

## Formulas

### 기본 데미지 공식 (Main Damage Formula)

```
final_damage = max(1, physical_damage + elemental_damage - defense)
```

여기서:

```
physical_damage = floor((base_damage + flat_bonus) × (1 + pct_increase) × crit_multiplier)
elemental_damage = floor(elemental_base × (1 + elemental_pct_increase) × crit_multiplier)
```

| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| base_damage | int | 5-500 | 무기 데이터 | 무기 기본 공격력 |
| flat_bonus | int | 0-200 | 캐릭터 스탯 + 아이템 플랫 옵션 합산 | 고정 추가 데미지 |
| pct_increase | float | 0.0-3.0 | 아이템 % 옵션 합산 (예: 0.15 = 15%) | 데미지 증가율 |
| crit_multiplier | float | 1.0 or 1.5-3.75 | 크리티컬 미발동 시 1.0, 발동 시 배율 적용 | 크리티컬 배율 |
| elemental_base | int | 0-300 | 아이템 속성 데미지 옵션 | 속성 기본 데미지 |
| elemental_pct_increase | float | 0.0-2.0 | 아이템/스킬 속성 증가% | 속성 데미지 증가율 |
| defense | int | 0-400 | 피격자 방어력 (스탯 + 장비) | 고정 감소값 |

**Expected output range**:
- 초반 (Lv1, 노멀 장비): 5-15 데미지
- 중반 (Lv15, 레어 장비): 50-150 데미지
- 후반 (Lv30, 유니크 풀셋): 200-800 데미지
- 크리티컬 + 속성 시너지 빌드 최대: ~1500 데미지

### 크리티컬 판정 (Critical Hit)

```
is_critical = random(0.0, 1.0) < crit_chance
crit_multiplier = is_critical ? base_crit_damage × (1 + crit_damage_bonus) : 1.0
```

| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| crit_chance | float | 0.05-0.75 | 기본 5% + 아이템 옵션 합산 | 크리티컬 확률 (75% 캡) |
| base_crit_damage | float | 1.5 | 상수 | 기본 크리티컬 배율 |
| crit_damage_bonus | float | 0.0-1.5 | 아이템 옵션 합산 | 추가 크리티컬 데미지 (150% 캡) |

**최대 크리티컬 배율**: 1.5 × (1 + 1.5) = **3.75배** (풀빌드 극한)

### 스킬 데미지 (Skill Damage)

```
skill_damage = floor(final_damage × skill_multiplier)
```

| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| skill_multiplier | float | 0.5-5.0 | 스킬 데이터 | 스킬별 배율 (약한 다타 스킬 0.5x, 강한 단타 5.0x) |

스킬 데미지는 기본 데미지 공식의 결과에 스킬 배율을 곱한다. 속성 데미지도 포함.

### DoT 데미지 (Damage over Time)

```
dot_tick_damage = floor(dot_base × (1 + elemental_pct_increase))
```

DoT는 방어력 감소를 **적용하지 않는다** — 방어를 무시하는 대신 기본 수치가 낮다.
이렇게 하면 높은 방어력의 적에게도 DoT가 유효하여 속성 빌드의 가치가 유지된다.

| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| dot_base | int | 3-50 | 상태 효과 데이터 | 틱당 기본 데미지 |
| tick_interval | float | 0.5-1.0 | 상태 효과 데이터 | 초 단위 틱 간격 |
| duration | float | 2.0-5.0 | 상태 효과 데이터 | 지속 시간 (초) |

## Edge Cases

| Scenario | Expected Behavior | Rationale |
|----------|------------------|-----------|
| defense > physical + elemental | 최소 1 데미지 보장 | 아무리 강한 적이라도 때리면 깎여야 성장 동기 유지 |
| crit_chance > 0.75 | 0.75로 클램프 | 100% 크리티컬은 크리티컬의 의미를 없앰 |
| crit_damage_bonus > 1.5 | 1.5로 클램프 | 3.75배 이상은 밸런스 폭발 |
| pct_increase < 0 (디버프) | -0.9로 클램프 | 90% 이상 감소 시 데미지가 0 근접 → 재미 없음 |
| elemental_base = 0 (속성 없음) | elemental_damage = 0, 물리만 적용 | 속성 장비 없는 초반 정상 동작 |
| 다중 속성 동시 적용 | 각 속성별 별도 계산 후 합산 | 화염+독 유니크 같은 멀티 속성 빌드 지원 |
| DoT 중 같은 속성 DoT 재적용 | 타이머 리셋, 데미지는 높은 쪽 유지 | 컨셉에서 정의한 "맞을 때마다 타이머 리셋" 룰 |
| DoT 중 다른 속성 DoT 적용 | 각각 독립 유지 (중첩) | 화염+독 동시 DoT로 빌드 다양성 확보 |
| 레벨 1 플레이어 vs 방어력 0 몬스터 | 기본 공식 그대로 적용, defense=0 | 초반 튜토리얼 몬스터는 방어력 없음 |
| 환경 데미지 (독 늪 등) | defense 적용 안함, 고정 데미지 | 환경 데미지는 장비로 방어 불가 — 회피가 답 |

## Dependencies

| System | Direction | Nature |
|--------|-----------|--------|
| **Combat System** | depends on this | Hard — 모든 기본 공격/콤보의 데미지 산출 |
| **Skill System** | depends on this | Hard — 스킬 데미지 = final_damage × skill_multiplier |
| **Status Effect System** | depends on this | Hard — DoT 틱 데미지 계산 |
| **Monster/Enemy AI** | depends on this | Hard — 몬스터 공격 데미지 산출 |
| **Combat Feedback** | depends on this | Hard — 최종 데미지 값 + 크리티컬 여부 + 속성 타입 수신 |
| **Monster Scaling** | depends on this | Soft — 스케일링 결과가 defense/attack 입력값에 반영될 뿐, 계산기 자체와 직접 연동 아님 |
| **Item System** | this references | Soft — 아이템 스탯이 입력값에 반영되지만, 계산기가 직접 조회하지 않음 (호출자가 합산 전달) |
| **Character Growth** | this references | Soft — 레벨 스탯이 입력값에 반영되지만, 역시 호출자가 합산 전달 |

**Hard dependency**: 해당 시스템 없이는 이 시스템이 작동 불가
**Soft dependency**: 데이터를 제공받지만 없어도 기본값으로 동작 가능

## Tuning Knobs

| Parameter | Current Value | Safe Range | Effect of Increase | Effect of Decrease |
|-----------|--------------|------------|-------------------|-------------------|
| base_crit_damage | 1.5 | 1.2-2.0 | 크리티컬 한 방이 강해짐, 크리 빌드 가치 상승 | 크리티컬 임팩트 약화, 크리 빌드 매력 감소 |
| crit_chance_cap | 0.75 | 0.5-0.9 | 크리 확률 빌드 천장 상승, 안정적 DPS 빌드 가능 | 크리 빌드에 운 요소 증가 |
| crit_damage_bonus_cap | 1.5 | 0.5-2.0 | 크리 빌드 극한 배율 상승, 후반 인플레 위험 | 크리 빌드 천장 낮아짐 |
| min_damage | 1 | 1-5 | 방어 관통 최소값 증가, 약한 적에겐 무의미 | 1 미만은 0이 되므로 비추천 |
| pct_increase_clamp_min | -0.9 | -0.95 to -0.5 | 디버프 더 강해짐, 플레이어 무력감 증가 | 디버프 약해짐, 속성 저항 가치 감소 |
| dot_defense_bypass | true | true/false | DoT가 방어 무시 → 속성 빌드 가치 유지 | DoT에 방어 적용 → 고방어 적에 속성 빌드 무력화 |
| environmental_defense_bypass | true | true/false | 환경 데미지 고정 → 회피 강제 | 장비로 환경 데미지 감소 가능 → 긴장감 감소 |

## Visual/Audio Requirements

| Event | Visual Feedback | Audio Feedback | Priority |
|-------|----------------|---------------|----------|
| 일반 데미지 | 흰색 데미지 숫자 팝업 | 타격음 (물리) | MVP |
| 크리티컬 데미지 | 노란색 확대 숫자 팝업 + 화면 미세 흔들림 | 강화된 타격음 + 크리티컬 효과음 | MVP |
| 속성 데미지 (Fire) | 주황색 숫자 + 화염 파티클 | 화염 효과음 | Vertical Slice |
| 속성 데미지 (Ice) | 하늘색 숫자 + 빙결 파티클 | 빙결 효과음 | Vertical Slice |
| 속성 데미지 (Poison) | 녹색 숫자 + 독 파티클 | 독 효과음 | Vertical Slice |
| 속성 데미지 (Electric) | 보라색 숫자 + 전기 파티클 | 전기 효과음 | Vertical Slice |
| DoT 틱 | 작은 속성색 숫자 (일반보다 작게) | 약한 틱 효과음 | Vertical Slice |
| 최소 데미지 (1) | 회색 작은 숫자 | 약한 타격음 | MVP |
| 환경 데미지 | 빨간색 숫자 (경고 느낌) | 환경별 효과음 | Alpha |

## UI Requirements

| Information | Display Location | Update Frequency | Condition |
|-------------|-----------------|-----------------|-----------|
| 데미지 숫자 팝업 | 피격 대상 위 (월드 스페이스) | 매 타격 시 | 항상 |
| 크리티컬 표시 | 데미지 숫자에 크기/색상 차이 | 매 크리티컬 시 | 크리티컬 발동 |
| DPS 표시 | HUD (선택적) | 매 초 | 디버그/옵션 |

## Acceptance Criteria

- [ ] `calculate_damage()` 호출 시 물리 + 속성 데미지를 올바르게 합산한다
- [ ] defense가 총 데미지보다 커도 최소 1 데미지가 보장된다
- [ ] 크리티컬 확률 0.75 캡이 적용된다
- [ ] 크리티컬 배율 최대 3.75배가 초과되지 않는다
- [ ] pct_increase가 음수일 때 -0.9 클램프가 동작한다
- [ ] 속성 없는 공격 시 elemental_damage = 0으로 정상 계산된다
- [ ] 다중 속성 동시 적용 시 각각 별도 계산 후 합산된다
- [ ] DoT 데미지에 defense가 적용되지 않는다
- [ ] 환경 데미지에 defense가 적용되지 않는다
- [ ] 같은 속성 DoT 재적용 시 타이머만 리셋, 데미지는 높은 쪽 유지
- [ ] 다른 속성 DoT는 독립적으로 중첩된다
- [ ] 플레이어→몬스터, 몬스터→플레이어 양방향 동일 파이프라인 사용
- [ ] 스킬 배율(skill_multiplier) 적용 시 기본 공식 결과에 정확히 곱해진다
- [ ] Performance: 단일 calculate_damage 호출이 0.1ms 이내 완료
- [ ] 모든 수치(캡, 기본값, 배율)가 외부 데이터 파일에서 로드되며 하드코딩 없음

## Open Questions

| Question | Owner | Deadline | Resolution |
|----------|-------|----------|-----------|
| 방어력 관통(armor penetration) 옵션을 유니크 아이템에 추가할지? | game-designer | Item System 설계 시 | — |
| 속성 저항(elemental resistance)을 몬스터/플레이어에 추가할지? | game-designer | Status Effect System 설계 시 | — |
| 후반 데미지 인플레이션 제어를 위한 소프트캡 필요 여부 | systems-designer | Alpha 밸런싱 시 | — |
