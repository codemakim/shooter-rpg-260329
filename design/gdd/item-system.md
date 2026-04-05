# Item System

> **Status**: In Design
> **Author**: user + game-designer
> **Last Updated**: 2026-04-05
> **Implements Pillar**: "한 방의 쾌감" (Loot Dopamine)

## Overview

Item System은 게임 내 모든 아이템의 데이터 구조, 생성, 스탯 계산을 관리하는 Feature 레이어 시스템이다. Normal/Magic/Rare/Unique 4등급 체계로 아이템을 분류하고, Item Affix Generator가 생성한 랜덤 옵션을 부착한다. 무기, 방어구, 장신구 3종의 장비 카테고리를 정의하며, 아이템의 최종 스탯을 집계하여 Player Controller/Combat System에 제공한다.

## Player Fantasy

"'이 무기 존나 좋다'의 순간." 드랍된 레어 건블레이드의 옵션을 하나씩 읽는 순간 — +화염 데미지, +공격 속도, +크리티컬 확률. 지금 낀 것과 비교하며 "바꿀까 말까"를 고민하는 그 시간이 IRONCLAW의 중독 루프. 유니크 아이템은 이름만 봐도 심장이 뛰고, 스킬 변형 옵션을 읽으며 새 빌드를 구상하게 된다.

## Detailed Design

### Core Rules

1. **아이템 데이터 구조**:
   ```
   ItemData:
     id: String                    # 고유 ID ("gunblade_tier1")
     base_name: String             # 기본 이름 ("건블레이드")
     display_name: String          # 최종 표시명 ("불타는 건블레이드 속도의")
     rarity: String                # "normal" / "magic" / "rare" / "unique"
     item_level: int               # 아이템 레벨 (드랍 지역/몬스터 레벨로 결정)
     slot: String                  # "weapon" / "armor" / "accessory"
     base_stats: Dictionary        # 기본 스탯 {attack: 10, defense: 0, hp: 0...}
     affixes: Array[AffixInstance] # Item Affix Generator 결과
     unique_data: UniqueItemData   # Unique 전용 (null for others)
     icon: String                  # 아이콘 리소스 경로
     sell_price: int               # NPC 판매 가격
   ```

2. **4등급 체계**:

   | 등급 | 표시 색상 | Base Stats | Affixes | 특수 |
   |------|----------|-----------|---------|------|
   | **Normal** | 흰색 | 기본 | 0개 | — |
   | **Magic** | 파란색 | 기본 | 1~2개 | prefix/suffix 이름 조합 |
   | **Rare** | 노란색 | 기본 × 1.1 | 2~4개 | 랜덤 고유 이름 |
   | **Unique** | 주황색 | 고정 (사전 정의) | 고정 옵션 | 스킬 변형 슬롯, 로어 텍스트 |

3. **장비 슬롯 4종**:
   - **Weapon** (1개): 건블레이드 류. 기본 공격력, 공격 속도 결정
   - **Armor** (1개): 방어구. 방어력, HP 보너스
   - **Accessory 1** (1개): 장신구. 유틸 스탯 (CDR, 크리, 속성 데미지 등)
   - **Accessory 2** (1개): 장신구. 동일 종류도 장착 가능

4. **아이템 생성 흐름**:
   ```
   Drop Table → 등급 결정 + base item 선택 + item_level 설정
   → [Normal] base_stats만 적용
   → [Magic/Rare] Item Affix Generator.generate_affixes() 호출 → affixes 부착
   → [Rare] base_stats × 1.1 보정
   → [Unique] 사전 정의 unique_data 적용 (Affix Generator 우회)
   → display_name 생성 → sell_price 계산 → ItemData 완성
   ```

5. **스탯 집계**: 장착된 모든 아이템(최대 4개)의 base_stats + affixes를 합산. 같은 스탯 키는 additive 합산 후 Player Controller / Damage Calculator에 최종 보정치로 제공.

6. **아이템 비교**: 현재 장착 아이템과 비교 대상 아이템의 스탯 차이를 계산하여 UI에 ▲(상승)/▼(하락) 표시.

7. **UniqueItemData** (Unique 전용):
   ```
   UniqueItemData:
     fixed_stats: Dictionary      # 고정 스탯 (affix 대신)
     flavor_text: String          # 로어 텍스트
     skill_mutation: Dictionary   # {skill_id: mutation_data} — 스킬 변형 정보 (VS)
   ```

8. **드랍 시 즉시 확인**: 아이템 드랍 시 옵션이 즉시 공개. 미확인(unidentified) 시스템 없음.

### States and Transitions

아이템의 **소유 상태** (아이템 자체는 상태 머신 아님):

| State | 설명 | 전환 |
|-------|------|------|
| **Dropped** | 던전 바닥에 존재 (드랍 오브젝트) | 생성 시 |
| **Inventoried** | 인벤토리에 보관 중 | 플레이어 픽업 시 |
| **Equipped** | 장비 슬롯에 장착 중 | 인벤토리에서 장착 시 |
| **Sold** | NPC에 판매됨 → 제거 | 판매 확인 시 |

- **Dropped → Inventoried**: 플레이어가 드랍 오브젝트에 접촉하면 자동 픽업
- **Inventoried ↔ Equipped**: 장착/해제 (인벤토리 UI에서 조작)
- **Inventoried → Sold**: NPC 상점에서 판매 (골드 획득)
- **Dropped → 소멸**: 방 전환 시 미회수 드랍 아이템 자동 제거

### Interactions with Other Systems

| System | 방향 | 인터페이스 |
|--------|------|-----------|
| **Item Affix Generator** | ← 입력 | Magic/Rare 생성 시 `generate_affixes()` 호출 |
| **Drop Table** | ← 입력 | 드랍 등급, 아이템 레벨, base item ID 수신 |
| **Inventory System** | → 출력 | ItemData 전달, 소유/보관 관리 위임 |
| **Equipment Slot** | → 출력 | 장착 아이템 스탯 집계 제공 |
| **Damage Calculator** | → 출력 | 무기 base_damage + affix 보정 제공 |
| **Skill Mutation** | → 출력 | Unique의 skill_mutation 데이터 제공 (VS) |
| **HUD / Inventory UI** | → 출력 | 아이템 표시 정보 (이름, 등급 색상, 스탯, 아이콘) |
| **Save/Load** | ↔ 양방향 | 아이템 직렬화/역직렬화 (VS) |

## Formulas

### 1. Rare Base Stats 보정

```
rare_stat = floor(normal_stat × RARE_STAT_MULTIPLIER)
# RARE_STAT_MULTIPLIER = 1.1
```

### 2. 판매 가격

```
sell_price = base_price × rarity_multiplier + sum(affix_value_contribution)
```

| 등급 | rarity_multiplier |
|------|-------------------|
| Normal | 1.0 |
| Magic | 1.5 |
| Rare | 2.5 |
| Unique | 5.0 |

### 3. 스탯 집계

```
total_stats = {}
for each equipped_item in [weapon, armor, accessory_1, accessory_2]:
    for stat, value in item.base_stats:
        total_stats[stat] += value
    for affix in item.affixes:
        total_stats[affix.stat] += affix.rolled_value
# total_stats를 Player Controller / Damage Calculator에 전달
```

### 4. 아이템 비교

```
for each stat in union(current_item.stats, compared_item.stats):
    diff = compared_value - current_value
    # diff > 0: ▲ (녹색), diff < 0: ▼ (적색), diff == 0: 표시 안 함
```

## Edge Cases

1. **인벤토리 가득 찬 상태에서 픽업**: 픽업 실패 → 드랍 오브젝트 유지 + "인벤토리 가득" 알림 표시
2. **같은 Unique 중복 드랍**: 동일 Unique 여러 개 소유 가능 (D2 스타일)
3. **장비 슬롯 불일치**: weapon을 armor 슬롯에 장착 시도 → 거부, 에러 없음
4. **아이템 레벨 0**: base_stats만 있는 최약 Normal 아이템
5. **Accessory 동일 아이템 2개 장착**: 허용 — 같은 Unique 반지 2개 가능
6. **방 전환 시 드랍 아이템**: 미회수 아이템 전부 소멸. 보스 드랍은 자동 픽업으로 보호
7. **스탯 오버플로우**: Item System은 합산 스탯에 상한을 두지 않음 — 캡은 소비 시스템(Damage Calculator, Player Controller)이 처리
8. **장비 해제 시 스탯 미충족**: 다른 장비의 요구 스탯을 현재 장비가 제공하는 경우 없음 (요구 스탯 시스템 없음, MVP)

## Dependencies

### Upstream (이 시스템이 의존하는 시스템)

| System | 의존 유형 | 설명 |
|--------|-----------|------|
| **Item Affix Generator** | Hard | Magic/Rare 아이템 affix 생성 |

### Downstream (이 시스템에 의존하는 시스템)

| System | 의존 유형 | 설명 |
|--------|-----------|------|
| **Drop Table** | Hard | 드랍 시 아이템 생성 요청 |
| **Inventory System** | Hard | 아이템 보관/관리 |
| **Equipment Slot** | Hard | 장착 아이템 스탯 집계 |
| **Skill Mutation** | Soft (VS) | Unique의 skill_mutation 데이터 소비 |
| **Save/Load** | Soft (VS) | 아이템 직렬화/역직렬화 |
| **Damage Calculator** | Soft | 무기 스탯 참조 |

## Tuning Knobs

| Knob | 기본값 | 안전 범위 | 영향하는 게임플레이 | 극단값 효과 |
|------|--------|-----------|---------------------|-------------|
| `rare_stat_multiplier` | 1.1 | 1.0 ~ 1.3 | Rare vs Normal 기본 스탯 격차 | 1.0이면 차이 없음, 높으면 Rare 압도적 |
| `rarity_sell_multiplier` | N1.0/M1.5/R2.5/U5.0 | 등급별 | 골드 경제 밸런스 | 높으면 상위 등급 판매로 골드 인플레 |
| `equip_slot_count` | 4 | 3 ~ 6 | 빌드 깊이 vs 복잡도 | 적으면 단순, 많으면 스탯 인플레 |
| `auto_pickup_range` | 48px | 32 ~ 80px | 드랍 픽업 편의성 | 작으면 정확히 밟아야 함, 크면 자석처럼 |
| `boss_auto_pickup` | true | true/false | 보스 드랍 안전 보호 | false면 보스 드랍 미회수 가능 |

## Visual/Audio Requirements

- 등급별 드랍 오브젝트 색상: Normal(흰), Magic(파랑), Rare(노랑), Unique(주황) 글로우
- 드랍 시 등급에 따른 SFX: Normal(일반), Magic(약간 반짝), Rare(화려), Unique(레전더리 팡파르)
- Unique 드랍 시 화면 전체 골드 플래시 이펙트 + 특수 SFX
- 픽업 시 아이템이 캐릭터 쪽으로 빨려오는 애니메이션
- 드랍 오브젝트: 아이템 아이콘 + 등급 색상 배경 (미니 카드 형태)

## UI Requirements

- 아이템 툴팁: 이름(등급 색상) + base stats + affix 목록 + 비교(▲/▼)
- Unique 툴팁: 고정 옵션 + 스킬 변형 설명 + flavor text (이탤릭)
- 드랍 시 화면 우측에 간단한 드랍 알림 (아이콘 + 이름 + 등급 색상)
- Rare/Unique 드랍 시 화면 중앙에 잠깐 이름 표시 (축하 연출)

## Acceptance Criteria

1. Normal/Magic/Rare/Unique 4등급 아이템이 올바른 구조로 생성
2. Magic 아이템에 1~2개, Rare에 2~4개 affix 정상 부착
3. Unique 아이템이 고정 옵션 + skill_mutation 데이터를 포함
4. 장착된 4개 아이템의 스탯이 정확히 합산되어 캐릭터 스탯에 반영
5. 아이템 비교 시 ▲/▼ 표시가 모든 스탯에 대해 정확
6. 드랍 오브젝트 접촉 시 자동 픽업 (인벤토리 여유 시)
7. 인벤토리 가득 시 픽업 거부 + "인벤토리 가득" 알림
8. Rare의 base_stats가 Normal 대비 ×1.1 보정 적용
9. 방 전환 시 미회수 드랍 아이템 정상 제거
10. 보스 드랍 자동 인벤토리 전송

## Open Questions

1. **아이템 강화/업그레이드**: Normal→Magic 등 등급 업그레이드 메카닉 필요 여부 — Economy 밸런스에서 결정
2. **세트 아이템**: 디아블로식 세트 아이템(2개 이상 장착 시 보너스) 추가 여부 — Vertical Slice에서 검토
3. **아이템 레벨 스케일링 커브**: 지역별 아이템 레벨 범위와 base_stats 성장 곡선 — 밸런스 스프레드시트에서 정의
