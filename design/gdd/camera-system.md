# Camera System

> **Status**: In Design
> **Author**: user + game-designer
> **Last Updated**: 2026-04-05
> **Implements Pillar**: "20분의 밀도" (Dense Runs), "한 손의 몰입" (Effortless Control)

## Overview

Camera System은 탑다운 뷰 카메라를 관리하는 Foundation 시스템이다. 플레이어 캐릭터를 중심으로 부드럽게 추적하며, 전투 시 화면 흔들림(screenshake), PC에서 마우스 방향 카메라 리드(look-ahead), 방 전환 시 스무스 트랜지션을 제공한다. 모바일 화면 크기에 맞춘 고정 줌을 기본으로 하며, 보스전 등 특수 상황에서 줌 아웃을 지원한다.

## Player Fantasy

"전장이 한눈에 들어온다." 어디서 적이 오는지, 내 스킬이 어디까지 닿는지 항상 명확하다. 강한 공격이 터지면 화면이 흔들리며 임팩트를 체감하고, 보스 등장 시 카메라가 자연스럽게 빠지며 위압감을 전달한다. 카메라는 공기처럼 — 존재를 의식하지 않지만, 없으면 모든 게 어색해지는 시스템.

## Detailed Design

### Core Rules

1. **기본 추적**: Camera2D가 플레이어 캐릭터를 부드럽게 추적한다 (lerp 기반 스무딩).

2. **카메라 리드 (Look-Ahead)**:
   - 모바일: 이동 방향 벡터 × `LEAD_DISTANCE`만큼 카메라 중심 오프셋
   - PC: 마우스 방향(aim) 벡터 × `LEAD_DISTANCE`만큼 카메라 중심 오프셋
   - 정지 시 리드 부드럽게 해제, 캐릭터 중심으로 복귀

3. **화면 흔들림 (Screenshake)**:
   - 3 파라미터: 강도(intensity), 지속시간(duration), 감쇠 커브(decay)
   - 여러 흔들림 동시 요청 시 가장 강한 것 기준 합산 (additive, `MAX_SHAKE_INTENSITY` 캡)
   - 크리티컬 히트: 약한 흔들림 (intensity 0.3), 보스 피격: 중간 (0.6), 보스 사망: 강한 (1.0)

4. **줌 제어**:
   - 기본 줌: 고정값 — 모바일 화면에서 던전 방이 적절히 보이는 수준
   - 보스전 진입 시 줌 아웃 (보스 + 플레이어가 화면에 동시 표시)
   - 줌 전환은 lerp로 부드럽게 (`ZOOM_TRANSITION_SPEED`)

5. **방 전환 트랜지션**: 던전 방 이동 시 카메라가 새 방 중심으로 스무스 이동

6. **방 경계 클램핑**: 카메라가 현재 방의 경계 밖을 비추지 않도록 뷰포트 영역을 방 경계 내로 제한

### States and Transitions

| State | 설명 | 전환 조건 |
|-------|------|-----------|
| **Following** | 기본 상태. 플레이어 추적 + 리드 적용 | 기본 |
| **Transitioning** | 방 전환 중. 이전 위치 → 새 방 목표 위치로 이동 | 방 전환 시그널 수신 시 |
| **Boss Zoom** | 줌 아웃 상태. 보스 아레나 전체 표시 | 보스전 진입 시 |
| **Cutscene** | 스크립트 제어. 지정 좌표로 이동/줌 | 컷신 시작 시 |
| **Shake** | Following/Boss Zoom과 병행. 오프셋 추가 | shake 이벤트 수신 시 |

- **Shake**는 독립 레이어 — Following, Boss Zoom 위에 오프셋으로 적용 (상태가 아닌 오버레이)
- **Transitioning** 완료 후 자동으로 Following 복귀
- **Boss Zoom** 종료 시 부드럽게 Following으로 복귀 (줌 + 위치 동시 전환)
- **Cutscene** 종료 시 Following으로 복귀

### Interactions with Other Systems

| System | 방향 | 인터페이스 |
|--------|------|-----------|
| **Player Controller** | ← 입력 | 플레이어 위치, 이동 방향 벡터 수신 |
| **Input System** | ← 입력 | PC에서 aim 벡터 수신 (카메라 리드 계산용) |
| **Combat System** | ← 입력 | 히트/크리티컬/사망 이벤트 수신 → screenshake 트리거 |
| **Dungeon Generator** | ← 입력 | 현재 방 경계 Rect2 데이터 수신 (클램핑용) |
| **Scene Manager** | ← 입력 | 방 전환 시그널 수신 → Transitioning 상태 진입 |
| **HUD** | → 출력 | 카메라 뷰포트 정보 전달 → UI 레이아웃 기준 |

## Formulas

### 1. 카메라 추적 (Lerp)

```
target_pos = player_pos + lead_offset
camera_pos = camera_pos.lerp(target_pos, FOLLOW_SMOOTHING * delta)
camera_pos = clamp_to_room_bounds(camera_pos, room_rect, viewport_size)
```

| Variable | 설명 | 기본값 | 범위 |
|----------|------|--------|------|
| `FOLLOW_SMOOTHING` | 추적 반응성 (높을수록 빠름) | 8.0 | 4.0 ~ 15.0 |

### 2. 리드 오프셋

```
# 모바일
target_lead = move_direction * LEAD_DISTANCE
# PC
target_lead = aim_direction * LEAD_DISTANCE

current_lead = current_lead.lerp(target_lead, LEAD_SMOOTHING * delta)
lead_offset = current_lead
```

| Variable | 설명 | 기본값 | 범위 |
|----------|------|--------|------|
| `LEAD_DISTANCE` | 리드 최대 거리 (px) | 40 | 20 ~ 80 |
| `LEAD_SMOOTHING` | 리드 전환 부드러움 | 5.0 | 2.0 ~ 10.0 |

### 3. Screenshake 오프셋

```
shake_offset = Vector2(randf_range(-1, 1), randf_range(-1, 1)) * current_intensity
current_intensity = initial_intensity * decay_curve.sample(elapsed / duration)
final_camera_pos = camera_pos + shake_offset
```

| Variable | 설명 | 기본값 | 범위 |
|----------|------|--------|------|
| `initial_intensity` | 요청 시 흔들림 강도 (px) | 이벤트별 | 1 ~ 10 |
| `duration` | 흔들림 지속 시간 (s) | 이벤트별 | 0.1 ~ 0.5 |
| `decay_curve` | 감쇠 커브 | ease-out | — |
| `MAX_SHAKE_INTENSITY` | 합산 최대 강도 캡 (px) | 10 | 5 ~ 20 |

### 4. 줌 전환

```
current_zoom = current_zoom.lerp(target_zoom, ZOOM_SPEED * delta)
```

| Variable | 설명 | 기본값 | 범위 |
|----------|------|--------|------|
| `ZOOM_SPEED` | 줌 전환 속도 | 3.0 | 1.0 ~ 8.0 |

## Edge Cases

1. **방 경계보다 뷰포트가 큰 경우**: 방 중심에 카메라 고정, 클램핑 비활성화
2. **보스가 아레나 밖으로 이탈**: 플레이어 기준 Following으로 폴백
3. **동시 shake 다수 요청**: 합산 intensity가 `MAX_SHAKE_INTENSITY`(10px) 초과 시 캡 적용
4. **방 전환 중 피격**: Transitioning 중에도 shake 오버레이 정상 적용
5. **플레이어 사망 시**: 사망 위치에서 카메라 정지, 약간 줌 인 (사망 연출)
6. **극단적 이동 속도 (대시)**: 리드 오프셋에 `LEAD_DISTANCE` 캡이 적용되어 과도한 리드 방지

## Dependencies

### Upstream (이 시스템이 의존하는 시스템)

없음 — Foundation 레이어. 독립 동작.

### Downstream (이 시스템에 의존하는 시스템)

| System | 의존 유형 | 설명 |
|--------|-----------|------|
| **Player Controller** | Hard | 카메라 뷰포트 정보 필요 (조준, 화면 좌표 변환) |
| **HUD** | Soft | 카메라 뷰포트 기준으로 UI 레이아웃 배치 |

## Tuning Knobs

| Knob | 기본값 | 안전 범위 | 영향하는 게임플레이 | 극단값 효과 |
|------|--------|-----------|---------------------|-------------|
| `follow_smoothing` | 8.0 | 4.0 ~ 15.0 | 카메라 추적 반응성 | 낮으면 늘어지는 느낌, 높으면 딱딱한 느낌 |
| `lead_distance` | 40px | 20 ~ 80px | 전방 시야 확보량 | 낮으면 시야 좁음, 높으면 캐릭터가 화면 구석으로 |
| `lead_smoothing` | 5.0 | 2.0 ~ 10.0 | 리드 전환 부드러움 | 낮으면 리드 변경 시 울렁임, 높으면 반응 느림 |
| `max_shake_intensity` | 10px | 5 ~ 20px | 최대 화면 흔들림 | 낮으면 임팩트 약함, 높으면 어지러움 |
| `default_zoom` | Vector2(1, 1) | 0.8 ~ 1.2 | 기본 시야 범위 | 낮으면 넓은 시야 (적 작음), 높으면 좁은 시야 |
| `boss_zoom` | Vector2(0.75, 0.75) | 0.6 ~ 0.9 | 보스전 시야 범위 | 낮으면 보스가 매우 작음, 높으면 화면 밖으로 |
| `zoom_transition_speed` | 3.0 | 1.0 ~ 8.0 | 줌 전환 속도 | 낮으면 느린 연출, 높으면 갑작스러움 |
| `room_transition_speed` | 5.0 | 2.0 ~ 10.0 | 방 전환 카메라 속도 | 낮으면 긴 전환, 높으면 즉시 이동 느낌 |

## Visual/Audio Requirements

- 카메라 자체는 비주얼/오디오 없음 (투명 시스템)
- screenshake 시 화면 가장자리에 미세한 비네팅 효과 (선택 사항, 폴리시 단계)
- 방 전환 시 짧은 페이드 또는 와이프 효과 (Scene Manager 담당, 카메라는 위치 이동만)

## UI Requirements

- 설정 메뉴에 "화면 흔들림" ON/OFF 토글 (접근성)
- screenshake OFF 시 모든 흔들림 비활성 — 흔들림 민감 플레이어 배려

## Acceptance Criteria

1. 플레이어 이동 시 카메라가 부드럽게 추적 (끊김/지터 없음)
2. 이동 방향으로 카메라 리드가 적용되어 전방 시야 확보
3. 정지 시 리드가 부드럽게 해제되어 캐릭터 중심 복귀
4. 크리티컬 히트 시 약한 screenshake, 보스 사망 시 강한 screenshake 발생
5. 보스전 진입 시 줌 아웃, 종료 시 줌 복귀 (전환 부드러움)
6. 방 전환 시 새 방으로 카메라 스무스 이동 (순간 이동 아님)
7. 카메라가 현재 방 경계 밖을 비추지 않음 (클램핑 검증)
8. 설정에서 screenshake OFF 시 흔들림 완전 비활성
9. Camera System 프레임당 처리 시간 0.3ms 이내

## Open Questions

1. **사망 연출 줌 인 정도**: 프로토타입에서 줌 인 비율과 지속 시간 테스트 필요
2. **방 전환 이펙트**: 페이드 vs 와이프 vs 하드컷 — Scene Manager GDD에서 결정
3. **모바일 기기별 해상도 대응**: 기본 줌 값을 화면 비율에 따라 자동 조정할지 여부
