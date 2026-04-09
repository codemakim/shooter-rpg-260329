# Drop Table

> **Status**: In Design
> **Author**: user + game-designer
> **Last Updated**: 2026-04-08
> **Implements Pillar**: "한 방의 쾌감" (Loot Dopamine)

## Overview

Drop Table은 던전 내 모든 아이템 드랍의 **확률표**와 **품질 결정** 로직을 관리하는 Feature 레이어 시스템이다. 몬스터 처치, 보스 처치, 상자 개방, 지역 클리어 등의 드랍 이벤트가 발생하면 해당 소스에 연결된 Drop Table을 조회하여 (1) 드랍 여부, (2) 드랍 개수, (3) 아이템 등급(Normal/Magic/Rare/Unique), (4) base item 선택, (5) item_level을 결정한 뒤 Item System에 생성을 요청한다. 지역별 item_level 범위, 몬스터 티어별 등급 가중치, 보스 고유 드랍 풀, Magic Find(MF) 보정 등 디아블로2의 드랍 로직을 축약 재현하는 것이 목적이다.

## Player Fantasy

"이번 보스가 뭘 뱉을까?" — 보스의 HP 바가 바닥날 때 손가락이 떨리는 그 순간. 일반 몹은 대부분 쓰레기를 뱉지만, 가끔 터지는 노란색 섬광(Rare). 보스 방에 들어가기 전 심호흡, 처치 후 바닥에 쏟아지는 아이템들을 눈으로 훑으며 주황색 글로우(Unique)를 찾는 탐색의 긴장감. "엘리트 몹 풀이 좋다더라" "3지역 상자가 Rare 확률 높다더라" 같은 커뮤니티 지식이 쌓이는 시스템 — 드랍의 **예측 가능성**과 **도박성**의 균형이 IRONCLAW 루핑의 엔진.

## Detailed Design

### Core Rules

1. **Drop Source 분류**:

   | Source Type | 예시 | Drop Table 할당 방식 |
   |-------------|------|---------------------|
   | **Normal Mob** | 일반 잡몹 | 몬스터 종류별 공용 테이블 (지역+티어 기반) |
   | **Elite Mob** | 푸른/노란색 네임드 | 일반 테이블 + 등급 가중치 보너스 |
   | **Boss** | 지역 보스 | 전용 테이블 (고정 드랍 + 랜덤 풀) |
   | **Chest** | 방 내부 상자 | 지역별 공용 상자 테이블 |
   | **Room Clear** | 방 완전 소탕 보상 | 지역별 클리어 보너스 테이블 |

2. **DropTable 데이터 구조**:
   ```
   DropTable:
     id: String                    # "region2_mob_tier1"
     rolls: int                    # 1회 드랍 시도 수 (보스는 3~5회)
     no_drop_weight: float         # "아무것도 안 나옴" 가중치
     entries: Array[DropEntry]     # 드랍 후보 목록
     item_level_range: Vector2i    # (min, max) item_level 결정 범위
     guaranteed_drops: Array       # 보스용 고정 드랍 (항상 출현)

   DropEntry:
     rarity_weights: Dictionary    # {normal: 100, magic: 30, rare: 8, unique: 1}
     base_item_pool: Array[String] # 후보 base item ID 목록
     slot_filter: Array            # ["weapon", "armor", "accessory"] — 결과 slot 제한
     entry_weight: float           # 이 entry가 선택될 가중치
   ```

3. **드랍 결정 흐름**:
   ```
   Drop Event 수신 (source, source_level, MF_bonus)
   → DropTable 조회 (source ID 기반)
   → guaranteed_drops 즉시 생성 (있다면)
   → rolls 횟수만큼 반복:
     → no_drop_weight vs total_entry_weight 가중 추출
     → [no_drop 선택] skip
     → [entry 선택]
        → rarity_weights에 MF 보정 적용 → 등급 결정
        → base_item_pool에서 slot_filter 통과 아이템 균일 랜덤 선택
        → item_level = randi_range(item_level_range.x, item_level_range.y)
        → Item System.create_item(base_item_id, rarity, item_level) 호출
   → 생성된 ItemData 배열을 Drop Spawner에 전달 (드랍 오브젝트 생성)
   ```

4. **지역별 item_level 스케일링** (5개 지역 기준):

   | Region | Name | Mob item_level | Boss item_level | Main Rarity |
   |--------|------|---------------|-----------------|-------------|
   | 1 | 황무지 외곽 | 1~5 | 5~8 | Normal/Magic |
   | 2 | 방사능 습지 | 5~12 | 10~15 | Magic |
   | 3 | 붕괴 도시 | 12~20 | 18~24 | Magic/Rare |
   | 4 | 지하 연구소 | 20~30 | 28~35 | Rare |
   | 5 | 균열의 중심 | 30~40 | 38~45 | Rare/Unique |

5. **Magic Find (MF) 보정**: 장비/스킬로 얻는 MF 스탯. Normal 가중치를 낮추고 상위 등급 가중치를 증가. CAP 400% (5배 보정). 공식은 아래 Formulas 참조.

6. **Unique 드랍 특수 규칙**:
   - Unique는 **기본 weight가 매우 낮음** (1 vs Normal 100)
   - **Unique Pool**: 지역별 등장 가능한 Unique 목록 사전 정의. 드랍 결정 시 pool에서 균일 추출
   - **중복 허용**: 같은 Unique 여러 개 드랍 가능 (D2 스타일)
   - **보스 전용 Unique**: 특정 보스는 guaranteed_drops에 고정 Unique 포함 (낮은 확률)

7. **엘리트 몹 보너스**: 엘리트 몬스터는 일반 테이블을 공유하되 `rolls × 2`, `rarity_weights`에 `magic × 2, rare × 3, unique × 5` 보정.

8. **드랍 위치**: 드랍된 아이템은 몬스터 사망 위치 주변에 부채꼴로 분산. 같은 지점 중첩 방지 (최소 16px 간격). 보스 드랍은 보스 중심 원형 분포 + auto-pickup 대상(Item System).

### States and Transitions

**Stateless 데이터 조회 모듈**. Drop Event 발생 시 테이블 조회 → 결과 반환. 자체 상태 없음.

```
roll_drops(source_id: String, source_level: int, mf_bonus: float) → Array[ItemData]
```

- 입력: 드랍 소스 ID, 소스 레벨, 플레이어 MF 보너스
- 출력: 생성된 ItemData 배열 (0개 가능)
- 부작용: Item System에 create_item 호출 (Item System 측 부작용)

### Interactions with Other Systems

| System | 방향 | 인터페이스 |
|--------|------|-----------|
| **Item System** | → 출력 | `create_item(base_item_id, rarity, item_level, slot)` 호출 |
| **Monster/Enemy AI** | ← 입력 | 몬스터 사망 시 `roll_drops()` 트리거, monster_id + level 전달 |
| **Dungeon Progression** | ← 입력 | 방 클리어 보너스, 지역 정보(item_level 범위) 제공 |
| **Dungeon Generator** | ← 입력 | 상자 배치 시 상자 테이블 ID 할당 |
| **Equipment Slot** | ← 입력 | 플레이어 MF 스탯 수신 |
| **Combat System** | 간접 | 몬스터 사망 이벤트를 Monster AI 경유로 수신 |
| **Drop Spawner (Item System 내)** | → 출력 | 생성된 ItemData → 월드 드랍 오브젝트 배치 요청 |

## Formulas

### 1. MF 보정 등급 가중치

```
effective_mf = min(mf_bonus, MF_CAP)   # MF_CAP = 4.0 (400%)
adjusted_weights = {
    normal: base_weights.normal × (1 / (1 + effective_mf × 0.5)),
    magic:  base_weights.magic  × (1 + effective_mf × 0.6),
    rare:   base_weights.rare   × (1 + effective_mf × 1.0),
    unique: base_weights.unique × (1 + effective_mf × 1.5),
}
```

| Variable | 설명 | 범위 |
|----------|------|------|
| `mf_bonus` | 플레이어 MF 스탯 (0.0 = 0%, 1.0 = 100%) | 0.0 ~ 4.0 |
| `base_weights` | DropEntry의 기본 rarity_weights | 정수 |

**예시** (base: N=100, M=30, R=8, U=1):

| MF | Normal | Magic | Rare | Unique | Unique 확률 |
|----|--------|-------|------|--------|-------------|
| 0% | 100 | 30 | 8 | 1 | 0.72% |
| 100% | 66.7 | 48 | 16 | 2.5 | 1.91% |
| 200% | 50 | 66 | 24 | 4 | 2.77% |
| 400%(Cap) | 33.3 | 102 | 40 | 7 | 3.79% |

### 2. 가중 등급 롤

```
total = sum(adjusted_weights.values())
roll = randf() × total
cumulative = 0.0
for rarity, weight in adjusted_weights:
    cumulative += weight
    if roll <= cumulative:
        return rarity
```

### 3. 드랍 횟수

```
# 각 roll 당 독립 확률:
actual_drops = 0
for i in range(rolls):
    if weighted_pick([no_drop_weight, total_entry_weight]) == entry:
        actual_drops += 1
```

| Source | rolls | no_drop_weight | 평균 드랍 개수 |
|--------|-------|----------------|--------------|
| Normal Mob | 1 | 70 | ~0.3 |
| Elite Mob | 2 | 40 | ~1.2 |
| Boss | 5 | 10 | ~4.5 + guaranteed |
| Chest | 3 | 20 | ~2.4 |

### 4. item_level 결정

```
item_level = randi_range(table.item_level_range.x, table.item_level_range.y)
# 지역별 테이블이 item_level_range를 정의
```

### 5. 엘리트 보너스 적용

```
if is_elite:
    rolls *= 2
    rarity_weights.magic  *= 2
    rarity_weights.rare   *= 3
    rarity_weights.unique *= 5
```

## Edge Cases

1. **빈 base_item_pool**: 등급 결정 후 pool이 비어 있으면 해당 drop skip (크래시 없음, 로그 경고)
2. **slot_filter 불일치**: pool 내 모든 base item이 slot_filter 통과 못하면 해당 drop skip
3. **Unique Pool 고갈**: 지역 Unique pool이 비어 있으면 Rare로 등급 다운그레이드
4. **MF 음수값**: 0.0으로 클램프 (저주 효과로 MF 감소 시도 방지)
5. **rolls = 0**: guaranteed_drops만 생성. 정상 동작
6. **동시 대량 드랍 (화면에 30+ 드랍)**: 드랍 오브젝트는 Item System 측에서 제한 (최대 100개). 초과 시 가장 오래된 것부터 제거
7. **source_level이 item_level_range 초과**: 테이블의 range가 우선. source_level은 단순 정보용
8. **존재하지 않는 base_item_id**: 로그 에러 + 해당 drop skip. 게임 진행 방해 없음
9. **같은 프레임 다수 몬스터 사망**: 각각 독립 roll_drops 호출. 드랍 오브젝트는 위치 분산
10. **테이블 ID 미존재**: 기본 fallback 테이블 사용 (Normal 아이템 1개). 로그 경고

## Dependencies

### Upstream (이 시스템이 의존하는 시스템)

| System | 의존 유형 | 설명 |
|--------|-----------|------|
| **Item System** | Hard | `create_item()` 호출로 아이템 인스턴스 생성 |

### Downstream (이 시스템에 의존하는 시스템)

| System | 의존 유형 | 설명 |
|--------|-----------|------|
| **Monster/Enemy AI** | Hard | 몬스터 사망 시 `roll_drops()` 트리거 |
| **Dungeon Progression** | Hard | 방 클리어/지역 정보로 테이블 조회 |
| **Dungeon Generator** | Soft | 상자 배치 시 테이블 할당 |
| **Equipment Slot** | Soft | MF 스탯 제공 |

## Tuning Knobs

| Knob | 기본값 | 안전 범위 | 영향하는 게임플레이 | 극단값 효과 |
|------|--------|-----------|---------------------|-------------|
| `mf_cap` | 4.0 (400%) | 2.0 ~ 6.0 | MF 스탯 최대 효율 | 낮으면 MF 가치 ↓, 높으면 Unique 남발 |
| `mf_normal_penalty` | 0.5 | 0.3 ~ 0.8 | MF당 Normal 감소율 | 높으면 MF로 Normal 씨말림 |
| `mf_unique_bonus` | 1.5 | 1.0 ~ 2.5 | MF당 Unique 증가율 | 높으면 Unique 인플레이션 |
| `elite_rolls_multiplier` | 2 | 1 ~ 3 | 엘리트 드랍 개수 배율 | 높으면 엘리트 파밍 집중 |
| `elite_unique_multiplier` | 5 | 2 ~ 10 | 엘리트 Unique 가중치 | 높으면 엘리트 처치가 핵심 루프 |
| `base_unique_weight` | 1 | 0.5 ~ 3 | 기본 Unique 등장 빈도 | 낮으면 의미 없음, 높으면 흔해짐 |
| `boss_guaranteed_count` | 보스별 1~3 | 0 ~ 5 | 보스 확정 드랍 수 | 많으면 보스 의존 플레이 |
| 지역별 `item_level_range` | 테이블별 | — | 지역 성장 곡선 | 좁으면 동일 레벨 반복, 넓으면 스파이크 |
| `no_drop_weight` | source별 | 0 ~ 200 | 전체 드랍 밀도 | 높으면 건조, 낮으면 인벤토리 범람 |

## Visual/Audio Requirements

- 드랍 시 등급별 글로우/SFX는 Item System(Drop Spawner)에서 처리 — Drop Table은 데이터만 결정
- **Drop Table 로그 채널** (디버그): 개발 중 드랍 내역을 콘솔에 출력 (`roll=0.42, rarity=rare, item=gunblade_t2`)
- 보스 guaranteed drop은 보스 사망 이펙트와 함께 연출 (Item System 측)

## UI Requirements

- 드랍 테이블 자체는 UI 없음
- 디버그 오버레이(개발용): 현재 방의 예상 드랍 테이블 ID + rolls + item_level_range 표시
- (Polish) 플레이어 MF 툴팁에 "등급 가중치 보정: Normal -25%, Unique +150%" 같은 구체적 영향 표시

## Acceptance Criteria

1. 일반 몬스터 100회 처치 시 드랍 확률이 설정값에 통계적으로 수렴 (±5%)
2. 보스 처치 시 guaranteed_drops 100% 등장
3. 엘리트 몹이 일반 몹 대비 평균 드랍 수 2배 이상, Unique 확률 5배 이상
4. MF 0% vs 400% 비교 시 Unique 확률이 공식대로 증가 (1000회 샘플)
5. MF 400% 초과 값 입력 시 400%로 캡
6. 지역별 item_level이 설정 범위를 벗어나지 않음
7. 빈 pool / 존재하지 않는 base_item_id 시 크래시 없이 skip + 경고 로그
8. 같은 프레임 5개 몬스터 동시 사망 시 드랍 위치가 겹치지 않음 (최소 16px 간격)
9. Unique Pool 고갈 시 Rare로 다운그레이드, Unique 미생성
10. 1회 roll_drops 처리 시간 2ms 이내 (모바일 기준)

## Open Questions

1. **MF 스탯의 장비 분포**: MF affix를 Rare 이상에만 허용할지, 지역 특화 장비에만 배치할지 — Item Affix Generator 밸런스에서 결정
2. **시즌제 드랍 풀**: 로그라이트 구조상 시즌/주차별 드랍 테이블 로테이션 필요 여부 — Live-Ops에서 검토
3. **플레이어 레벨 vs item_level 보정**: 플레이어 레벨이 item_level_range보다 훨씬 높을 때 스케일링 보정 필요 여부 — Monster Scaling 시스템과 연동하여 결정
4. **드랍 희석 페널티**: 같은 방 반복 클리어 시 드랍률 감소(D2 rest-area penalty) 도입 여부 — Dungeon Progression에서 결정
