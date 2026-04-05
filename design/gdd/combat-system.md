# Combat System

> **Status**: In Design
> **Author**: user + game-designer
> **Last Updated**: 2026-04-05
> **Implements Pillar**: "한 방의 쾌감" (Loot Dopamine), "20분의 밀도" (Dense Runs)

## Overview

Combat System은 플레이어와 적 사이의 모든 전투 상호작용을 처리하는 Feature 레이어 시스템이다. Player Controller로부터 콤보/스킬 발동 요청을 받아 히트박스 생성, 투사체 발사, 히트 판정을 수행하고, 판정 결과를 Damage Calculator에 전달하여 최종 데미지를 산출한다. 적의 공격도 동일한 파이프라��을 사용한다. 건블레이드의 근접(슬래시)+원거리(사격) 하이브리드 전투가 핵심이며, 히트스톱/히���이펙트 등 전투 주스(juice)를 통해 타격감을 극대화한다.

## Player Fantasy

"베고 쏘고 터뜨린다." 건블레이드가 적을 베면 화면이 멈칫하고, 사격이 관통하면 연쇄 폭발이 일어난다. 장비가 강해질수록 슬래시 범위가 넓어지고 투사체가 늘어나며, 한 방에 적 무리가 쓸려나가는 '한 방의 쾌감'을 직접 체감한다. 전투는 빠르고, 화려하고, 치명적이어야 한다.

## Detailed Design

### Core Rules

1. **공격 타입 2종**:
   - **Melee (슬래시)**: 부채꼴 Area2D 히트박스. 조준 방향 기준 `SLASH_ARC`(120°) 범위, `SLASH_RANGE`(80px) 거리 내 모든 적 히트. 콤보 Slash1, Slash2에 사용
   - **Ranged (사격)**: 투사체 발사. 조준 방향으로 `PROJECTILE_SPEED`(400px/s) 이동. 기본 단일 타격(첫 적 히트 시 소멸). 관통은 장비 옵션으로 해금

2. **히트 판정 파이프라인**:
   ```
   공격 발동 → 히트박스/투사체 생성 → 충돌 감지(Area2D)
   → 히트 대상 수집 → Damage Calculator에 전달
   → 데미지 결과 수신 → 대상에 적용 + 이벤트 발송
   ```

3. **히트 이벤트 데이터**:
   ```
   HitEvent:
     attacker: Node          # 공격자 참조
     target: Node            # 피격자 참조
     damage_result: Dictionary  # Damage Calculator 반환값
     hit_position: Vector2   # 충돌 위치
     hit_direction: Vector2  # 피격 방향 (넉백 계산용)
     attack_type: String     # "melee" / "ranged" / "skill"
     is_critical: bool       # 크리티컬 여부
   ```

4. **히트스톱 (Hitstop)**:
   - 크리티컬 히트: 30ms 일시정지 (`Engine.time_scale = 0.0` → 복원)
   - 보스 피격: 50ms 일시정지
   - 일반 히트: 히트스톱 없음, 히트 이펙트만 재생

5. **적 공격 처리**: 적도 동일한 히트 판정 파이프라인 사용. Monster AI가 공격 요청 → Combat System이 히트박스/투사체 생성 → 플레이어 피격 판정 → Damage Calculator 처리

6. **다중 히트 방지**: 같은 공격 인스턴스에서 동일 대상은 1회만 피격. 공격별 `hit_targets: Array` 로 관리

7. **투사체 규칙**:
   - 수명: `PROJECTILE_LIFETIME`(3초) 후 자동 소멸
   - 벽 충돌: 즉시 소멸 (반사는 장비 옵션으로 해금)
   - 최대 동시 투사체: `MAX_PROJECTILES`(50) — 초과 시 가장 오래된 것 제거
   - 기본: 단일 타격 (첫 적 히트 시 소멸)
   - 관통(장비 옵션): `pierce_count`만큼 적 관통 후 소멸

### States and Transitions

Combat System 자체는 상태 머신이 아닌 **이벤트 드리븐** 시스템. 개별 공격 인스턴스의 라이프사이클:

| Phase | 설명 | 다음 단계 |
|-------|------|-----------|
| **Spawn** | 히트박스/투사체 생성, 오브젝트 풀에서 가져옴 | Active |
| **Active** | 히트 판정 수행 중 (슬래시: 1~2프레임, 투사체: 이동 중) | Hit / Expire |
| **Hit** | 대상 적중, Damage Calculator에 전달 | Destroy (단일) / Active (관통) |
| **Expire** | 수명 종료 또는 범위 초과 | Destroy |
| **Destroy** | 인스턴스 비활성화, 오브젝트 풀에 반환 | — |

- 슬래시 히트박스: Spawn → Active(1~2프레임) → Destroy (즉시 소멸)
- 투사체: Spawn → Active(이동 중) → Hit(단일)/Expire → Destroy
- 관통 투사체: Hit 후 pierce_count 감소, 0이 되면 Destroy

### Interactions with Other Systems

| System | 방향 | 인터페이스 |
|--------|------|-----------|
| **Player Controller** | ← 입력 | 콤보 단계(Slash1/Slash2/Shot) + 타겟 방향 → 공격 생성 요청 |
| **Skill System** | ← 입력 | 스킬 공격 데이터(히트박스 형태, 투사체 패턴 등) → 히트박스/투사체 생성 |
| **Damage Calculator** | → 출력 | 히트 데이터(공격자 스탯, 대상 스탯, 속성) 전달 → 최종 데미지 결과 수신 |
| **Status Effect System** | → 출력 | 속성 히트 시 상태이상 적용 요청 (화염 → 화상, 빙결 → 냉기 등) |
| **Monster/Enemy AI** | ← 입력 | 적 공격 요청(공격 타입, 방향, 스탯) 수신 → 히트박스/투사체 생성 |
| **Combat Feedback** | → 출력 | 히트 이벤트 시그널 발송 → 데미지 넘버, 이펙트, SFX 트리거 |
| **Camera System** | → 출력 | 크리티컬/보스 히트 시 screenshake 요청 |
| **Equipment Slot** | ← 입력 | 무기 스탯(범위, 투사체 수, 관통 여부, 공격 속도) 수신 |

## Formulas

### 1. 슬래시 히트 판정

```
for each enemy in range(SLASH_RANGE):
    angle = abs(aim_direction.angle_to(direction_to_enemy))
    if angle <= SLASH_ARC / 2:
        register_hit(enemy)
```

| Variable | 설명 | 기본값 | 범위 |
|----------|------|--------|------|
| `SLASH_RANGE` | 슬래시 도달 거리 (px) | 80 | 50 ~ 120 |
| `SLASH_ARC` | 슬래시 부채꼴 각도 | 120° | 60° ~ 180° |

### 2. 투사체 이동

```
projectile.position += projectile.direction × PROJECTILE_SPEED × delta
if lifetime_elapsed >= PROJECTILE_LIFETIME:
    destroy_projectile()
```

| Variable | 설명 | 기본값 | 범위 |
|----------|------|--------|------|
| `PROJECTILE_SPEED` | 투사체 이동 속도 (px/s) | 400 | 200 ~ 600 |
| `PROJECTILE_LIFETIME` | 투사체 최대 수명 (초) | 3.0 | 1.0 ~ 5.0 |

### 3. 히트스톱 타이밍

| 트리거 | 지속 시간 | Engine.time_scale | 비고 |
|--------|-----------|-------------------|------|
| 일반 히트 | 0ms | 1.0 | 이펙트만 |
| 크리티컬 히트 | 30ms | 0.0 | 완전 정지 |
| 보스 피격 | 50ms | 0.0 | 완전 정지 |
| 보스 사망 | 200ms | 0.1 | 슬로모션 |

### 4. 오브젝트 풀 설정

| Pool | 초기 할당 | 최대 |
|------|-----------|------|
| 슬래시 히트박스 | 10 | 20 |
| 플레이어 투사체 | 30 | 50 |
| 적 투사체 | 30 | 50 |

## Edge Cases

1. **동시 다수 히트 (슬래시로 10+마리)**: 모든 적에 개별 Damage Calculator 호출. 성능 보호: 한 프레임 최대 `MAX_HITS_PER_FRAME`(20) 히트 처리
2. **히트스톱 중 추가 히트**: 히트스톱 큐잉 — 현재 히트스톱 종료 후 다음 실행, 최대 2개 큐
3. **투사체가 방 경계 도달**: 벽 콜리전으로 즉시 소멸. 방 전환 시 모든 활성 투사체 제거
4. **무적 프레임 중 히트**: 히트 판정 성공하지만 데미지 0 처리. 이펙트 미표시
5. **보스 CC 면역 중 상태이상 히트**: 데미지 정상 적용, 상태이상만 무시 (Status Effect System 처리)
6. **오브젝트 풀 고갈**: 풀 부족 시 가장 오래된 활성 인스턴스 강제 회수 + 경고 로그
7. **자기 자신 히트**: attacker == target인 경우 무시. 아군/적 판정은 콜리전 레이어 마스크로 분리
8. **0 데미지 히트**: Damage Calculator가 0 반환해도 히트 이벤트 발생 (넉백은 적용, 이펙트는 미니멀)
9. **다중 투사체 동시 히트 (같은 적)**: 각 투사체는 독립 공격 인스턴스이므로 모두 개별 적용
10. **관통 투사체 + 보스**: 관통 투사체도 보스는 1회만 히트 (hit_targets 목록으로 관리)

## Dependencies

### Upstream (이 시스템이 의존하는 시스템)

| System | 의존 유형 | 설명 |
|--------|-----------|------|
| **Player Controller** | Hard | 콤보 단계 + 타겟 방향 → 공격 생성 요청 |
| **Damage Calculator** | Hard | 히트 데이터 전달 → 최종 데미지 결과 수신 |
| **Status Effect System** | Soft | 속성 히트 시 상태이상 적용 요청 |
| **Skill System** | Soft | 스킬 공격 데이터(히트박스 형태, 투사체 패턴) 수신 |
| **Monster/Enemy AI** | Soft | 적 공격 요청(공격 타입, 방향, 스탯) 수신 |
| **Equipment Slot** | Soft | 무기 스탯(범위, 관통, 투사체 수 등) 수신 |

### Downstream (이 시스템에 의존하는 시스템)

| System | 의존 유형 | 설명 |
|--------|-----------|------|
| **Combat Feedback** | Soft | 히트 이벤트 발송 → 데미지 넘버, 이펙트, SFX |
| **Camera System** | Soft | 크리티컬/보스 히트 시 screenshake 요청 |
| **HUD** | Soft | 전투 중 HP 변동, 데미지 표시 |
| **Audio Manager** | Soft | 히트/슬래시/사격 SFX 이벤트 |

## Tuning Knobs

| Knob | 기본값 | 안전 범위 | 영향하는 게임플레이 | 극단값 효과 |
|------|--------|-----------|---------------------|-------------|
| `slash_range` | 80px | 50 ~ 120 | 근접 공격 도달 범위 | 짧으면 위험한 근접전, 넓으면 안전하지만 긴장감 ↓ |
| `slash_arc` | 120° | 60° ~ 180° | 슬래시 부채꼴 각도 | 좁으면 정밀, 넓으면 광역 — 120°가 적정 |
| `projectile_speed` | 400 px/s | 200 ~ 600 | 사격 속도감/회피 가능성 | 느리면 적이 회피, 빠르면 즉발에 가까움 |
| `projectile_lifetime` | 3.0초 | 1.0 ~ 5.0 | 사격 실질 사거리 | 짧으면 근거리만, 길면 화면 밖까지 |
| `hitstop_critical` | 30ms | 15 ~ 60ms | 크리티컬 타격감 | 짧으면 체감 안 됨, 길면 전투 끊김 |
| `hitstop_boss` | 50ms | 30 ~ 100ms | 보스 피격 타격감 | 상동 |
| `max_projectiles` | 50 | 30 ~ 100 | 화면 내 투사체 밀도 | 낮으면 화력 제한, 높으면 성능 부담 |
| `max_hits_per_frame` | 20 | 10 ~ 30 | 프레임 내 처리량 | 낮으면 대규모 전투 누락, 높으면 스파이크 |

## Visual/Audio Requirements

- **슬래시**: 무기 방향 부채꼴 슬래시 이펙트 (반원형 칼날 궤적, 1~2프레임)
- **사격**: 총구 화염 + 투사체 스프라이트 이동 + 피격 시 히트 파티클
- **크리티컬 히트**: 큰 히트 이펙트 + 화면 번쩍임(flash) + 확대된 데미지 넘버
- **히트스톱**: time_scale 조절 중 히트 이펙트 프리즈 (시각적으로 강조)
- **보스 사망**: 슬로모션 중 폭발 이펙트 시퀀스
- **SFX**: 슬래시 휘두르기, 총 발사, 히트(일반/크리/보스), 투사체 소멸 — 각각 별도 SFX 슬롯

## UI Requirements

- 데미지 넘버: 히트 위치에서 떠오르는 텍스트 (Combat Feedback 담당)
- HP 바 변동: 피격 시 즉시 반영 (HUD 담당)
- 크리티컬 표시: 데미지 넘버 크기/색상 차이 (Combat Feedback 담당)

## Acceptance Criteria

1. 슬래시 공격이 부채꼴(120°, 80px) 범위 내 모든 적에 동시 히트
2. 투사체가 첫 적에 맞으면 소멸 (기본), 관통 옵션 시 pierce_count만큼 관통
3. 크리티컬 히트 시 30ms 히트스톱이 체감 가능
4. 보스 사망 시 200ms 슬로모션 연출 정상 작동
5. 적의 공격도 동일 파이프라인으로 플레이어에게 데미지 적용
6. 같은 공격 인스턴스에서 같은 적이 2회 피격되지 않음
7. 방 전환 시 모든 활성 투사체/히트박스 정상 제거
8. 10마리 동시 히트 시 60fps 유지 (프레임 드랍 없음)
9. 오브젝트 풀 고갈 시 크래시 없이 가장 오래된 인스턴스 재활용
10. 무적 프레임 중 히트 시 데미지 0, 이펙트 미표시
11. 속성 공격 히트 시 Status Effect System에 상태이상 적용 요청 정상 전달

## Open Questions

1. **대시/회피 무적**: 대시 중 무적 프레임을 Combat System에서 처리할지, Player Controller에서 처리할지 — Skill System GDD에서 결정
2. **아군 투사체 충돌**: 소환수(Hivecaller)의 투사체가 플레이어 투사체와 충돌하는지 — 콜리전 레이어 설계 시 결정
3. **환경 오브젝트 파괴**: 폭발 통, 함정 등 환경 오브젝트도 Combat System 파이프라인을 사용할지 — Dungeon Generator GDD에서 결정
4. **원거리 적 투사체 반사**: 특정 스킬/장비로 적 투사체를 반사하는 메카닉 가능 여부 — Skill Mutation GDD에서 결정
