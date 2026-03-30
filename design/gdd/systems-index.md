# Systems Index: IRONCLAW

> **Status**: Draft
> **Created**: 2026-03-29
> **Last Updated**: 2026-03-29
> **Source Concept**: design/gdd/game-concept.md

---

## Overview

IRONCLAW는 탑다운 슈터 ARPG로, 소울나이트의 직관적 조작과 디아블로2의 루팅/빌드 시스템을
결합한다. 핵심 루프는 "던전 진입 → 전투 → 드랍 → 장비 강화 → 더 강한 던전" 이며,
이를 구동하는 28개 시스템이 식별되었다. 설계는 4개 필러(한 방의 쾌감, 20분의 밀도,
한 손의 몰입, 황무지의 색)를 기준으로 판단한다.

---

## Systems Enumeration

| # | System Name | Category | Priority | Status | Design Doc | Depends On |
|---|-------------|----------|----------|--------|------------|------------|
| 1 | Damage Calculator | Gameplay | MVP | Not Started | — | — |
| 2 | Status Effect System | Gameplay | MVP | Not Started | — | Damage Calculator |
| 3 | Input System | Core | MVP | Not Started | — | — |
| 4 | Camera System | Core | MVP | Not Started | — | — |
| 5 | Scene Manager | Core | MVP | Not Started | — | — |
| 6 | Player Controller | Core | MVP | Not Started | — | Input System, Camera System |
| 7 | Combat System | Gameplay | MVP | Not Started | — | Player Controller, Damage Calculator, Status Effect |
| 8 | Skill System | Gameplay | MVP | Not Started | — | Player Controller, Damage Calculator, Status Effect |
| 9 | Item Affix Generator | Economy | MVP | Not Started | — | — |
| 10 | Item System | Economy | MVP | Not Started | — | Item Affix Generator |
| 11 | Drop Table | Economy | MVP | Not Started | — | Item System |
| 12 | Inventory System | Economy | MVP | Not Started | — | Item System |
| 13 | Equipment Slot System | Economy | MVP | Not Started | — | Item System, Inventory System |
| 14 | Monster/Enemy AI | Gameplay | MVP | Not Started | — | Damage Calculator, Status Effect |
| 15 | Dungeon Generator | Gameplay | MVP | Not Started | — | Scene Manager, Monster AI |
| 16 | Dungeon Progression | Gameplay | MVP | Not Started | — | Dungeon Generator, Drop Table |
| 17 | HUD | UI | MVP | Not Started | — | Character Growth, Skill System, Combat System |
| 18 | Combat Feedback | UI | MVP | Not Started | — | Combat System, Damage Calculator |
| 19 | Skill Mutation System | Gameplay | Vertical Slice | Not Started | — | Skill System, Item System |
| 20 | Character Growth | Progression | Vertical Slice | Not Started | — | Player Controller |
| 21 | Monster Scaling | Gameplay | Vertical Slice | Not Started | — | Character Growth, Monster AI |
| 22 | Town/Lobby System | Gameplay | Vertical Slice | Not Started | — | Scene Manager, Inventory System |
| 23 | Save/Load | Persistence | Vertical Slice | Not Started | — | Inventory, Character Growth, Equipment Slot |
| 24 | Inventory UI | UI | Vertical Slice | Not Started | — | Inventory System, Equipment Slot |
| 25 | Town UI | UI | Vertical Slice | Not Started | — | Town/Lobby System |
| 26 | Class System | Gameplay | Alpha | Not Started | — | Skill System, Equipment Slot |
| 27 | Audio Manager | Audio | Alpha | Not Started | — | Combat System |
| 28 | Tutorial/Onboarding | Meta | Full Vision | Not Started | — | Combat, Skill, Inventory, HUD |

---

## Categories

| Category | Description |
|----------|-------------|
| **Core** | Foundation systems everything depends on — input, camera, scene management |
| **Gameplay** | Systems that make the game fun — combat, AI, dungeons, skills |
| **Economy** | Resource creation and consumption — items, loot, inventory, equipment |
| **Progression** | How the player grows over time — leveling, stats |
| **Persistence** | Save state and continuity — save/load |
| **UI** | Player-facing information displays — HUD, inventory screen, menus |
| **Audio** | Sound and music — BGM, SFX |
| **Meta** | Systems outside the core game loop — tutorials, onboarding |

---

## Priority Tiers

| Tier | Definition | Target | Systems |
|------|------------|--------|---------|
| **MVP** | Core loop 검증에 필수. "이게 재미있는가?" 테스트 | 4-6주 | 18개 |
| **Vertical Slice** | 완전한 한 세션 경험 (마을 → 던전 → 성장 → 반복) | 2-3개월 | 7개 |
| **Alpha** | 전체 기능 러프 (다중 클래스, 오디오) | 4-6개월 | 2개 |
| **Full Vision** | 폴리시, 온보딩 | 8-12개월 | 1개 |

---

## Dependency Map

### Foundation Layer (no dependencies)

1. **Input System** — 모든 플레이어 조작의 추상화 레이어
2. **Scene Manager** — 마을 ↔ 던전, 방 간 전환 관리
3. **Camera System** — 탑다운 뷰 추적, 화면 흔들림
4. **Damage Calculator** — 전투의 수학적 기반 (공격력, 방어력, 속성, 크리티컬)
5. **Item Affix Generator** — 아이템 랜덤 옵션 생성 데이터 모듈

### Core Layer (depends on Foundation)

6. **Player Controller** — depends on: Input System, Camera System
7. **Status Effect System** — depends on: Damage Calculator
8. **Item System** — depends on: Item Affix Generator

### Feature Layer (depends on Core)

9. **Combat System** — depends on: Player Controller, Damage Calculator, Status Effect
10. **Skill System** — depends on: Player Controller, Damage Calculator, Status Effect
11. **Monster/Enemy AI** — depends on: Damage Calculator, Status Effect
12. **Inventory System** — depends on: Item System
13. **Equipment Slot System** — depends on: Item System, Inventory System
14. **Drop Table** — depends on: Item System
15. **Character Growth** — depends on: Player Controller
16. **Dungeon Generator** — depends on: Scene Manager, Monster AI
17. **Dungeon Progression** — depends on: Dungeon Generator, Drop Table
18. **Skill Mutation System** — depends on: Skill System, Item System
19. **Monster Scaling** — depends on: Character Growth, Monster AI
20. **Class System** — depends on: Skill System, Equipment Slot
21. **Town/Lobby System** — depends on: Scene Manager, Inventory System
22. **Save/Load** — depends on: Inventory, Character Growth, Equipment Slot

### Presentation Layer (depends on Features)

23. **Combat Feedback** — depends on: Combat System, Damage Calculator
24. **HUD** — depends on: Character Growth, Skill System, Combat System
25. **Inventory UI** — depends on: Inventory System, Equipment Slot
26. **Town UI** — depends on: Town/Lobby System
27. **Audio Manager** — depends on: Combat System (event hooks)

### Polish Layer

28. **Tutorial/Onboarding** — depends on: Combat, Skill, Inventory, HUD

---

## Recommended Design Order

| Order | System | Priority | Layer | Est. Effort |
|-------|--------|----------|-------|-------------|
| 1 | Damage Calculator | MVP | Foundation | S |
| 2 | Status Effect System | MVP | Core | S |
| 3 | Input System | MVP | Foundation | S |
| 4 | Camera System | MVP | Foundation | S |
| 5 | Scene Manager | MVP | Foundation | S |
| 6 | Player Controller | MVP | Core | M |
| 7 | Combat System | MVP | Feature | L |
| 8 | Skill System | MVP | Feature | M |
| 9 | Item Affix Generator | MVP | Foundation | M |
| 10 | Item System | MVP | Feature | L |
| 11 | Drop Table | MVP | Feature | S |
| 12 | Inventory System | MVP | Feature | M |
| 13 | Equipment Slot System | MVP | Feature | S |
| 14 | Monster/Enemy AI | MVP | Feature | L |
| 15 | Dungeon Generator | MVP | Feature | L |
| 16 | Dungeon Progression | MVP | Feature | M |
| 17 | HUD | MVP | Presentation | M |
| 18 | Combat Feedback | MVP | Presentation | M |
| 19 | Skill Mutation System | Vertical Slice | Feature | M |
| 20 | Character Growth | Vertical Slice | Feature | M |
| 21 | Monster Scaling | Vertical Slice | Feature | S |
| 22 | Town/Lobby System | Vertical Slice | Feature | M |
| 23 | Save/Load | Vertical Slice | Feature | M |
| 24 | Inventory UI | Vertical Slice | Presentation | M |
| 25 | Town UI | Vertical Slice | Presentation | M |
| 26 | Class System | Alpha | Feature | L |
| 27 | Audio Manager | Alpha | Presentation | S |
| 28 | Tutorial/Onboarding | Full Vision | Polish | M |

Effort: S = 1 session, M = 2-3 sessions, L = 4+ sessions

---

## Circular Dependencies

- **Skill Mutation ↔ Item System**: 스킬 변형은 유니크 아이템 데이터에 의존하고, 유니크
  아이템의 가치는 스킬 변형에 의존. **해결**: 아이템 시스템이 "스킬 변형 데이터 구조"를
  제공하고, 스킬 변형 시스템이 이를 소비하는 단방향으로 설계. 인터페이스(데이터 구조)를
  아이템 시스템 GDD에서 먼저 정의.

---

## High-Risk Systems

| System | Risk Type | Risk Description | Mitigation |
|--------|-----------|-----------------|------------|
| Combat System | Design | 건블레이드 자동 콤보가 실제로 재미있는가? 30초 루프의 핵심. | MVP 프로토타입에서 즉시 검증 |
| Item Affix Generator | Technical | 랜덤 옵션 조합의 데이터 구조가 복잡해질 수 있음 | 데이터 드리븐 설계, 스프레드시트 관리 |
| Skill Mutation System | Design | 유니크 아이템 × 스킬 조합의 밸런스 폭발 | 변형 카테고리를 제한적으로 유지, 수치는 외부 설정 |
| Dungeon Generator | Technical | 방 랜덤 조합의 품질과 다양성 확보 | 방 풀을 충분히 크게, 규칙 기반 필터링 |
| Monster/Enemy AI | Scope | 지역별 고유 몬스터 + 보스 패턴 = 볼륨 큼 | MVP는 2-3종 기본 AI만, 보스 1체만 |

---

## Progress Tracker

| Metric | Count |
|--------|-------|
| Total systems identified | 28 |
| Design docs started | 0 |
| Design docs reviewed | 0 |
| Design docs approved | 0 |
| MVP systems designed | 0/18 |
| Vertical Slice systems designed | 0/7 |

---

## Next Steps

- [ ] Review and approve this systems enumeration
- [ ] Design MVP-tier systems first (use `/design-system [system-name]`)
- [ ] Run `/design-review` on each completed GDD
- [ ] Run `/gate-check pre-production` when MVP systems are designed
- [ ] Prototype the highest-risk system early (`/prototype core-combat`)
