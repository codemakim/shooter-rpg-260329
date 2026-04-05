# Skill System

> **Status**: In Design
> **Author**: user + game-designer
> **Last Updated**: 2026-04-05
> **Implements Pillar**: "한 방의 쾌감" (Loot Dopamine), "20분의 밀도" (Dense Runs)

## Overview

Skill System은 플레이어의 3개 액티브 스킬 슬롯을 관리하는 Feature 레이어 시스템이다. 각 스킬은 쿨다운 기반(마나 없음)으로 작동하며, 클래스별 고유 스킬 풀에서 장착한다. 스킬 발동 시 Combat System에 공격 생성을 요청하거나, Status Effect를 직접 적용하거나, 버프/유틸리티 효과를 실행한다. Skill Mutation System(Vertical Slice)과 연동하여 유니크 아이템이 스킬의 동작을 변형하는 것이 IRONCLAW 빌드 시스템의 핵심이다.

## Player Fantasy

"스킬 하나로 판을 뒤집는다." 쿨다운이 찼을 때 누르는 한 방 — 화면을 뒤덮는 화염, 적 무리를 얼리는 빙결파, 그림자 속에서 나타나는 연쇄 암살. 유니크 아이템을 장착하면 스킬이 완전히 다른 형태로 변형되고, "이 아이템 × 이 스킬" 조합을 발견하는 순간이 IRONCLAW의 진짜 득템이다.

## Detailed Design

### Core Rules

1. **스킬 슬롯**: 3개 액티브 슬롯. 클래스별 4개 스킬 풀에서 3개를 선택 장착. 스킬 교체는 마을에서만 가능. 슬롯에 타입 제한 없음 — 어떤 스킬이든 어떤 슬롯에든 장착 가능.

2. **스킬 데이터 구조**:
   ```
   SkillData:
     id: String                  # 고유 식별자 (e.g., "ironclad_slam")
     name: String                # 표시 이름
     class: String               # 소속 클래스
     description: String         # 설명 텍스트
     cooldown: float             # 기본 쿨다운 (초)
     damage_multiplier: float    # 기본공격 대비 데미지 배율
     element: String             # "none" / "fire" / "ice" / "poison" / "electric"
     effect_type: String         # "damage" / "buff" / "utility"
     hitbox_shape: String        # "arc" / "circle" / "line" / "projectile" / "self"
     hitbox_params: Dictionary   # 형태별 파라미터 (range, arc, width, count 등)
     cast_time: float            # 시전 시간 (0이면 즉시 발동)
     allow_move: bool            # 시전 중 이동 가능 여부
     interruptible: bool         # 피격 시 캔슬 가능 여부
     animation: String           # 스킬 애니메이션 키
     vfx: String                 # 이펙트 프리셋 키
     sfx: String                 # 사운드 프리셋 키
     mutation_slots: Array       # Skill Mutation 적용 가능 슬롯 (VS에서 사용)
   ```

3. **쿨다운 관리**:
   - 스킬 효과 적용 완료 시 쿨다운 타이머 시작
   - 실제 쿨다운: `actual_cooldown = base_cooldown × (1 - cooldown_reduction)`
   - CDR(쿨다운 감소) 캡: 50% — 최소 쿨다운 = base × 0.5
   - 쿨다운은 실시간 진행 (CC 중, 시전 중, 방 전환 중에도 감소)

4. **스킬 발동 흐름**:
   ```
   Input(skill_N press) → Skill System 수신 → 쿨다운 체크
   → [Ready] → cast_time 체크
   → [cast_time > 0] → Casting 상태 진입, cast_time 후 효과 적용
   → [cast_time == 0] → 즉시 효과 적용
   → Combat System에 공격 생성 요청 OR Status Effect 적용 OR 버프 적용
   → 쿨다운 시작
   ```

5. **스킬 타겟팅**: Player Controller의 자동조준 타겟 방향 사용. 스킬별 `hitbox_shape`에 따라:
   - `arc`: 타겟 방향 부채꼴
   - `circle`: 캐릭터 중심 원형 (방향 무관)
   - `line`: 타겟 방향 직선
   - `projectile`: 타겟 방향 투사체 발사
   - `self`: 자기 자신 (버프 스킬)

6. **스킬 레벨**: MVP에서는 모든 스킬 Lv1 고정. Character Growth(VS)에서 스킬 레벨업 시스템 추가 검토.

### States and Transitions

각 스킬 슬롯은 독립적으로 상태를 관리한다:

| State | 설명 | 전환 조건 |
|-------|------|-----------|
| **Ready** | 쿨다운 완료, 발동 가능 | cooldown_remaining ≤ 0 |
| **Casting** | 시전 중 (cast_time > 0인 스킬) | 스킬 발동 요청 + Ready 상태 |
| **Cooldown** | 쿨다운 진행 중, 발동 불가 | 스킬 효과 적용 완료 시 |

- 3개 슬롯이 각각 독립 상태 → 슬롯1 Cooldown 중에도 슬롯2 Ready 가능
- **Casting 중 피격**: `interruptible == true`이면 캔슬 (쿨다운 미적용, Ready 복귀)
- **Casting 중 CC(스턴/빙결)**: 무조건 캔슬 (쿨다운 미적용)
- **CC 중 발동 시도**: 무시 (Input System이 Disabled 상태이므로 도달하지 않음)
- **Cooldown → Ready**: 자동 전환 (cooldown_remaining이 0에 도달)

### Interactions with Other Systems

| System | 방향 | 인터페이스 |
|--------|------|-----------|
| **Player Controller** | ← 입력 | skill_N 발동 요청 + 현재 타겟 방향 수신 |
| **Combat System** | → 출력 | 공격형 스킬: hitbox_shape + hitbox_params + damage_multiplier 전달 |
| **Damage Calculator** | → 출력 | 스킬 데미지 계산 요청 (배율, 속성 포함) |
| **Status Effect System** | → 출력 | 버프/디버프 적용 요청 (자가 버프 or 적 대상) |
| **Skill Mutation** | ← 입력 | 유니크 아이템에 의한 SkillData 필드 오버라이드 수신 (VS) |
| **Equipment Slot** | ← 입력 | CDR, 스킬 데미지 보너스, 속성 부여 등 스탯 수신 |
| **HUD** | → 출력 | 슬롯별 스킬 아이콘, 쿨다운 잔여시간, Ready 상태 여부 |
| **Class System** | ← 입력 | 현재 클래스의 사용 가능 스킬 목록 수신 (Alpha) |

## Formulas

### 1. 스킬 데미지

```
skill_base = base_damage × damage_multiplier
# 이후 Damage Calculator 표준 파이프라인 적용:
# skill_base에 (1 + skill_damage_bonus) 곱한 값을 base_damage로 전달
```

| Variable | 설명 | 범위 |
|----------|------|------|
| `base_damage` | 캐릭터 기본 공격력 | 장비/레벨 의존 |
| `damage_multiplier` | 스킬별 데미지 배율 | 1.5 ~ 5.0× |
| `skill_damage_bonus` | 장비 스킬 데미지 보너스 | 0.0 ~ 1.0 |

### 2. 실제 쿨다운

```
actual_cooldown = base_cooldown × (1 - min(CDR, CDR_CAP))
# CDR_CAP = 0.5
```

| 예시 | base_cooldown | CDR | actual_cooldown |
|------|---------------|-----|-----------------|
| 초반 | 8초 | 0% | 8.0초 |
| 중반 | 8초 | 30% | 5.6초 |
| 후반(캡) | 8초 | 50%+ | 4.0초 |

### 3. 실제 시전 시간

```
actual_cast = cast_time × (1 - min(cast_speed_bonus, CAST_SPEED_CAP))
# CAST_SPEED_CAP = 0.5
```

- MVP 대부분 스킬: cast_time = 0 (즉시 발동)
- 강력한 스킬: cast_time 0.3 ~ 0.5초

## Edge Cases

1. **3스킬 동시 Ready + 같은 프레임 입력**: Input System FIFO 룰에 따라 skill_1 우선. 한 프레임 1스킬만 발동
2. **Casting 중 다른 스킬 입력**: 무시 (큐잉 안 함). Casting 완료 후 재입력 필요
3. **스킬 발동 직후 방 전환**: 스킬 효과 정상 적용, 쿨다운 정상 시작. 투사체는 방 전환 시 제거(Combat System)
4. **CDR 50% 초과 장비 조합**: CDR 합산 50% 초과해도 50%로 캡. HUD에서 캡 상태 표시
5. **스킬 교체 시 쿨다운**: 마을에서 스킬 교체 시 모든 쿨다운 리셋 (Ready)
6. **빈 슬롯**: 스킬 미장착 슬롯 입력 무시. 최소 1개 스킬 강제 장착
7. **Mutation이 hitbox_shape 변경**: Combat System에 전달하는 데이터만 변경, Skill System 로직은 동일
8. **쿨다운 중 스킬 교체 시도**: 마을에서만 교체 가능이므로 던전 내에서는 불가

## Dependencies

### Upstream (이 시스템이 의존하는 시스템)

| System | 의존 유형 | 설명 |
|--------|-----------|------|
| **Player Controller** | Hard | skill_N 발동 요청 + 타겟 방향 수신 |
| **Damage Calculator** | Hard | 스킬 데미지 계산 파이프라인 |
| **Status Effect System** | Soft | 버프/디버프 스킬의 효과 적용 |
| **Equipment Slot** | Soft | CDR, 스킬 데미지 보너스, 속성 부여 |
| **Skill Mutation** | Soft (VS) | 유니크 아이템에 의한 SkillData 오버라이드 |
| **Class System** | Soft (Alpha) | 클래스별 사용 가능 스킬 목록 |

### Downstream (이 시스템에 의존하는 시스템)

| System | 의존 유형 | 설명 |
|--------|-----------|------|
| **Combat System** | Hard | 스킬의 히트박스/투사체 생성 데이터 전달 |
| **HUD** | Soft | 스킬 아이콘, 쿨다운 잔여시간, Ready 상태 |

## Tuning Knobs

| Knob | 기본값 | 안전 범위 | 영향하는 게임플레이 | 극단값 효과 |
|------|--------|-----------|---------------------|-------------|
| `cdr_cap` | 0.5 (50%) | 0.3 ~ 0.7 | 최대 쿨다운 감소량 | 낮으면 CDR 스탯 가치 ↓, 높으면 스킬 스팸 |
| `cast_speed_cap` | 0.5 (50%) | 0.3 ~ 0.7 | 최대 시전 가속 | 상동 |
| `skills_per_class` | 4 | 3 ~ 6 | 빌드 다양성 | 적으면 선택지 없음, 많으면 밸런스 부담 |
| `max_equipped` | 3 | 2 ~ 4 | 동시 사용 스킬 수 | 적으면 단조로움, 많으면 버튼 부족(모바일) |
| 각 스킬 `cooldown` | 스킬별 | 3 ~ 15초 | 스킬 사용 빈도 | 짧으면 스킬 위주 플레이, 길면 기본공격 위주 |
| 각 스킬 `damage_multiplier` | 스킬별 | 1.5 ~ 5.0× | 스킬 vs 기본공격 가치 | 낮으면 스킬 무의미, 높으면 기본공격 무의미 |

## Visual/Audio Requirements

- 각 스킬은 고유 VFX 프리셋 (이펙트 스프라이트/파티클)
- 스킬 발동 시 캐릭터 전용 시전 애니메이션 (또는 범용 시전 포즈)
- 스킬별 고유 SFX (발동음 + 히트음)
- Casting 중인 스킬: 캐릭터 주변 시전 이펙트 (차징 느낌)
- 스킬 Ready 상태: 버튼 글로우/하이라이트로 사용 가능 표시

## UI Requirements

- 스킬 버튼 3개: 아이콘 + 쿨다운 오버레이 (시계방향 sweep)
- 쿨다운 중: 어두운 오버레이 + 남은 시간 숫자
- Ready 시: 버튼 밝아짐 + 미세한 펄스 효과
- 마을 스킬 교체 UI: 4개 스킬 풀 표시, 드래그 or 탭으로 슬롯 배치

## Acceptance Criteria

1. 3개 스킬 슬롯이 독립적으로 쿨다운 관리 (동시 다른 상태 가능)
2. Ready 상태에서 skill press 시 즉시 발동 (cast_time 0 스킬)
3. CDR 50% 캡이 정상 적용 (초과분 무시)
4. 쿨다운이 CC(스턴/빙결) 중에도 계속 감소
5. Casting 중 피격 시 interruptible 스킬은 캔슬, 쿨다운 미적용
6. 스킬 발동 시 Combat System에 정확한 hitbox 데이터 전달
7. 마을에서 스킬 교체 후 모든 쿨다운 리셋
8. 빈 슬롯 입력 시 에러 없이 무시
9. HUD에 스킬 아이콘 + 쿨다운 잔여시간 실시간 표시
10. Input System 버퍼 연동: 쿨다운 종료 60ms 전 입력이 즉시 발동

## Open Questions

1. **패시브 스킬**: 액티브 외에 패시브 스킬 슬롯이 필요한지 — Character Growth/Class System에서 결정
2. **궁극기(Ultimate)**: 4번째 특수 슬롯(게이지 기반 궁극기)의 필요성 — Vertical Slice에서 검토
3. **스킬 시너지**: 특정 스킬 조합 시 보너스 효과(e.g., 빙결 → 화염 = 증발 보너스) — 밸런스 테스트 후 결정
