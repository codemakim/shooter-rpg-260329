# Damage Calculator

> **Status**: In Design
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

[To be designed]

### States and Transitions

[To be designed]

### Interactions with Other Systems

[To be designed]

## Formulas

[To be designed]

## Edge Cases

[To be designed]

## Dependencies

[To be designed]

## Tuning Knobs

[To be designed]

## Visual/Audio Requirements

[To be designed]

## UI Requirements

[To be designed]

## Acceptance Criteria

[To be designed]

## Open Questions

[To be designed]
