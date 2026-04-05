# Item Affix Generator

> **Status**: In Design
> **Author**: user + game-designer
> **Last Updated**: 2026-04-05
> **Implements Pillar**: "한 방의 쾌감" (Loot Dopamine)

## Overview

Item Affix Generator는 아이템의 랜덤 옵션(접두사/접미사)을 생성하는 데이터 모듈이다. 디아블로2의 매직/레어 아이템처럼, 아이템 등급에 따라 정해진 개수의 affix를 affix 풀에서 뽑아 부여한다. 이 시스템은 순수 데이터 생성기로, 아이템 자체의 관리(인벤토리, 장착)는 Item System이 담당한다. "같은 무기라도 완전히 다른 옵션"이 나오는 루팅의 핵심 엔진.

## Player Fantasy

"이번엔 뭐가 붙었지?" 드랍된 아이템을 확인하는 순간의 두근거림. '불타는 빠른 건블레이드'처럼 접두사+접미사 조합이 내 빌드에 딱 맞을 때의 쾌감. 같은 이름의 무기도 옵션에 따라 쓰레기가 되거나 보물이 되는 — 디아블로의 "identify" 순간을 재현하는 시스템.

## Detailed Design

### Core Rules

1. **Affix 데이터 구조** (외부 데이터 파일로 관리):
   ```
   AffixData:
     id: String              # "prefix_fire_damage"
     name: String            # "불타는" (prefix) / "속도의" (suffix)
     type: String            # "prefix" / "suffix"
     stat: String            # 영향 스탯 키 ("flat_fire_damage", "attack_speed" 등)
     value_range: Vector2    # (min, max) — 롤 범위
     item_level_req: int     # 최소 아이템 레벨 (미만이면 풀에서 제외)
     weight: float           # 추출 가중치 (높을수록 자주 등장)
     allowed_slots: Array    # 장착 가능 슬롯 ["weapon", "armor", "accessory"]
     group: String           # 그룹 ID (같은 그룹 중복 방지)
     tier: int               # 같은 스탯의 강화 단계 (1=약, 3=강)
   ```

2. **등급별 Affix 개수**:

   | 등급 | Prefix | Suffix | 총 Affix | 비고 |
   |------|--------|--------|----------|------|
   | **Normal** | 0 | 0 | 0 | 기본 스탯만 |
   | **Magic** | 0~1 | 0~1 | 1~2 | 최소 1개 보장 |
   | **Rare** | 1~2 | 1~2 | 2~4 | 최소 2개 보장 |
   | **Unique** | — | — | 고정 | 사전 정의된 고정 옵션 + 스킬 변형 슬롯 |

3. **생성 흐름**:
   ```
   아이템 등급 결정 → affix 개수 롤 (등급별 범위 내)
   → prefix 개수 / suffix 개수 분배
   → 각각 유효 풀 필터링 (item_level, allowed_slots, group 중복 제거)
   → 가중치 기반 추출
   → value_range 내 균일 랜덤 롤
   → AffixInstance 생성
   ```

4. **AffixInstance** (생성 결과):
   ```
   AffixInstance:
     affix_id: String        # AffixData 참조 ID
     rolled_value: float     # 실제 롤된 수치
     display_name: String    # "불타는" 등 (아이템 이름 조합용)
   ```

5. **중복 방지**: 같은 `group`의 affix는 하나의 아이템에 1개만 부여. 같은 스탯의 다른 tier도 같은 group에 속하므로 공존 불가.

6. **아이템 레벨 필터**: `item_level_req`보다 낮은 아이템에는 해당 affix가 유효 풀에서 자동 제외.

7. **가중치 추출**: `weight` 기반 가중 랜덤 선택. 일반적 affix(+공격력 등)는 높은 weight, 강력한 affix(+크리티컬 데미지 등)는 낮은 weight.

8. **이름 조합 규칙**: `[Prefix] [Base Item Name] [Suffix]`
   - Magic: 1 prefix or 1 suffix → "불타는 건블레이드" or "건블레이드 속도의"
   - Rare: 랜덤 생성 이름 (D2 스타일, e.g., "황혼의 분노") — 접두사+접미사 이름 대신 고유 이름
   - Unique: 사전 정의 고유 이름

### States and Transitions

**Stateless 순수 함수형 모듈**. 외부 요청 시 affix를 생성하여 반환한다. 자체 상태 없음.

```
generate_affixes(item_rarity: String, item_level: int, item_slot: String) → Array[AffixInstance]
```

- 입력: 아이템 등급, 아이템 레벨, 장착 슬롯
- 출력: 생성된 AffixInstance 배열
- 부작용 없음 — 같은 입력에 대해 매번 다른 결과 (랜덤), 하지만 외부 상태 변경 없음

### Interactions with Other Systems

| System | 방향 | 인터페이스 |
|--------|------|-----------|
| **Item System** | ← 호출 | 아이템 생성 시 `generate_affixes()` 호출, 결과를 아이템에 부착 |
| **Drop Table** | 간접 | Drop Table이 등급/레벨 결정 → Item System이 해당 정보로 affix 생성 요청 |

## Formulas

### 1. Affix 개수 롤

```
# Magic: 1~2개 (최소 1 보장)
prefix_count = randi_range(0, 1)
suffix_count = randi_range(0, 1)
if prefix_count + suffix_count == 0:
    prefix_count = 1  # 최소 1개 보장

# Rare: 2~4개 (최소 2 보장)
prefix_count = randi_range(1, 2)
suffix_count = randi_range(1, 2)
```

### 2. 가중 랜덤 추출

```
total_weight = sum(affix.weight for affix in valid_pool)
roll = randf() × total_weight
cumulative = 0.0
for affix in valid_pool:
    cumulative += affix.weight
    if roll <= cumulative:
        return affix
```

### 3. 값 롤

```
rolled_value = randf_range(value_range.x, value_range.y)
# 정수 스탯 (+공격력 등): floor(rolled_value)
# 퍼센트 스탯 (+크리 확률 등): snapped(rolled_value, 0.1)
```

### 4. 유효 풀 필터링

```
valid_pool = affix_database.filter(
    type == target_type AND              # prefix or suffix
    item_level >= item_level_req AND     # 아이템 레벨 충족
    item_slot in allowed_slots AND       # 슬롯 호환
    group not in already_used_groups     # 그룹 중복 방지
)
```

## Edge Cases

1. **유효 풀 부족**: 요청 affix 수보다 유효 affix가 적으면 가능한 만큼만 생성. 0개도 허용
2. **모든 group 소진**: group 중복 제거 후 풀이 빈 경우 해당 타입(prefix/suffix) 추가 생성 중단
3. **value_range min == max**: 고정값 affix — 특정 티어에서 의도적으로 사용 가능
4. **아이템 레벨 0**: 모든 affix가 item_level_req를 충족하지 못해 affix 0개 → Normal과 동일
5. **Unique 아이템**: `generate_affixes()` 호출하지 않음 — Item System이 사전 정의된 고정 affix 직접 적용
6. **시드 기반 재현**: 세이브/로드 시 동일 아이템 재현을 위해 아이템별 RNG 시드 저장 가능

## Dependencies

### Upstream (이 시스템이 의존하는 시스템)

없음 — Foundation 레이어 데이터 모듈. 독립 동작.

### Downstream (이 시스템에 의존하는 시스템)

| System | 의존 유형 | 설명 |
|--------|-----------|------|
| **Item System** | Hard | 아이템 생성 시 `generate_affixes()` 호출하여 결과를 아이템에 부착 |

## Tuning Knobs

| Knob | 기본값 | 안전 범위 | 영향하는 게임플레이 | 극단값 효과 |
|------|--------|-----------|---------------------|-------------|
| `magic_affix_range` | [1, 2] | [1, 3] | Magic 아이템 옵션 수 | 적으면 단조, 많으면 Magic과 Rare 차이 흐려짐 |
| `rare_affix_range` | [2, 4] | [2, 6] | Rare 아이템 옵션 수 | 적으면 가치 낮음, 많으면 OP 아이템 빈발 |
| 각 affix `weight` | affix별 | 1 ~ 100 | 개별 affix 등장 빈도 | 높으면 흔함, 낮으면 희귀 |
| 각 affix `value_range` | affix별 | 스탯별 상이 | 수치 범위 — 밸런스 핵심 | 범위 넓으면 도박감 ↑, 좁으면 예측 가능 |
| 각 affix `item_level_req` | affix별 | 1 ~ 50 | 고급 affix 등장 시점 | 낮으면 초반부터 강력한 옵션, 높으면 후반에 집중 |

## Visual/Audio Requirements

- 이 시스템은 순수 데이터 모듈로 비주얼/오디오 없음
- affix에 의한 시각적 변화(무기 이펙트 색상 등)는 Item System / Equipment Slot에서 처리

## UI Requirements

- affix 표시 형식: 스탯 이름 + 수치 (e.g., "+15 화염 데미지", "+8% 공격 속도")
- affix 수치 품질 표시: 범위 대비 롤 품질을 색상으로 (하위 25% 회색, 상위 25% 금색 등) — 폴리시 단계
- 아이템 이름 조합: Magic은 "[Prefix] [Base] [Suffix]", Rare는 랜덤 고유명

## Acceptance Criteria

1. Magic 아이템에 1~2개 affix 정상 생성 (최소 1개 보장)
2. Rare 아이템에 2~4개 affix 정상 생성 (최소 2개 보장)
3. 같은 group affix가 하나의 아이템에 중복 부여되지 않음
4. `item_level_req`보다 낮은 아이템에 해당 affix 미등장
5. 가중치 높은 affix가 통계적으로 더 자주 등장 (1000회 생성 시 비율 검증)
6. `rolled_value`가 항상 `value_range` 내에 위치
7. Unique 아이템은 이 시스템 우회, 고정 옵션 정상 적용
8. 유효 풀 부족 시 크래시 없이 가능한 만큼만 생성
9. 생성 호출 1회당 처리 시간 1ms 이내

## Open Questions

1. **Affix 티어 구간**: 아이템 레벨 구간별 tier 1/2/3 등장 확률 커브 — 밸런스 스프레드시트에서 결정
2. **Rare 이름 생성**: 랜덤 고유 이름 풀의 크기와 생성 규칙 — Writer 에이전트에 위임 검토
3. **Affix 리롤 메카닉**: 마을 NPC에서 affix 재추첨 기능 필요 여부 — Economy 밸런스에서 결정
