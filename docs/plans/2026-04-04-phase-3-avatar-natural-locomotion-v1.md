# План: Phase 3 - natural locomotion, gait solver и body naturalness `v1`

Статус: DRAFT

## Цель

Довести avatar runtime после Phase 2 до более естественного поведения тела и походки для `self` и `remote` аватаров без отката к упрощенному `self-only` сценарию: убрать ощущение "голова и руки на палке", сократить foot skating, добавить предсказуемые переходы между `idle/walk/strafe/backpedal/turn`, и сохранить единый behavioral contract между локальным и удаленным отображением.

## Не-цель

- Не вводить новый production mesh/rig/avatar pack как обязательную зависимость фазы.
- Не переносить Phase 3 в lipsync, seating, customization или rich hand-tracking transport.
- Не переписывать transport/replay path из Phase 2, если текущий `room-state` и pose pipeline уже достаточно передают root/head/hands/locomotion.
- Не делать full cinematic leg fidelity для всех дистанций; дорогие улучшения допускаются только для `near avatars` с деградацией по `LOD`/distance.
- Не плодить отдельную логику natural locomotion для `self` и `remote`; базовая locomotion classification и transition logic должны оставаться общими.

## Предпосылки и ограничения

- Roadmap фиксирует Phase 3 как `Ноги, gait solver и body naturalness` в `docs/2026-04-01-noah-avatar-system-tz-roadmap.md:1336`, а не как новый asset/rig initiative.
- После предыдущих фаз уже существуют рабочие точки расширения в `apps/runtime-web/src/avatar/avatar-controller.ts`, `apps/runtime-web/src/avatar/avatar-animation.ts`, `apps/runtime-web/src/avatar/avatar-locomotion.ts`, `apps/runtime-web/src/avatar/avatar-ik.ts`, `apps/runtime-web/src/avatar/remote-avatar-runtime.ts`.
- В `apps/runtime-web/src/avatar/avatar-runtime.ts` и `packages/runtime-core/src/flags.ts` уже есть `avatarLegIkEnabled`; Phase 3 должна использовать существующий feature-flag entrypoint для безопасного rollout/rollback.
- Текущий pipeline procedural/stub уже умеет locomotion state, upper-body solve и remote pose ingest; значит Phase 3 должна эволюционно улучшать этот path, а не заменять его параллельным runtime.
- Для realism нельзя деградировать фазу до `self-only`: после закрытия Phase 2 это было бы функциональным ухудшением относительно уже имеющихся remote avatars.
- Локальная проверка недостаточна; изменения должны быть рассчитаны на staging verification и не ломать тяжелые scene rooms и existing avatar sync path.

## Подход

Сохранить текущий procedural/stub avatar pipeline как базу, расширить его до единого locomotion/body layer для `self` и `remote`, а дорогие улучшения ввести ступенчато. Сначала внедрить общий locomotion classifier, transition smoothing, pelvis/spine refinement, turn-in-place и anti-foot-skating contract. Затем добавить `near-avatar` refinement: более заметный gait pass, foot planting и `LOD`-ограничения, чтобы близкие аватары выглядели естественнее, а дальние оставались дешевыми и визуально стабильными. Для `remote` путь должен использовать те же правила выбора locomotion state и blend weights, но опираться на уже получаемый pose/root stream и существующую деградацию из Phase 2.

## Definition of Done

Фаза завершена, если:

1. `self` и `remote` используют одинаковый locomotion classifier для `idle/walk/strafe/backpedal/turn`, и на одинаковых траекториях не расходятся по базовой логике состояний.
2. При движении больше нет заметного body sliding без соответствующего gait motion, а резкие старты/остановки не создают грубого foot skating.
3. `turn-in-place` срабатывает предсказуемо и не вырождается в ложный шаг вперёд/вбок.
4. `near avatars` получают более естественный body/gait pass и `foot planting`, а `far avatars` деградируют мягко, без резких артефактов и без смены state machine.
5. При `avatarLegIkEnabled=false` runtime полностью возвращается к стабильному поведению Phase 2 без поломки avatars и room flow.
6. Локальный verification suite и `pnpm test:e2e:staging` зелёные, а staging smoke подтверждает отсутствие регрессий в `demo-room` и тяжёлых scene rooms.

## Задачи (чек-лист)

### 1. Зафиксировать Phase 3 contract

- [ ] Зафиксировать единый locomotion/body contract для `self` и `remote`: одинаковые состояния `idle/walk/strafe/backpedal/turn` и одинаковые базовые правила переходов.
- [ ] Зафиксировать границу фазы: работаем поверх текущего procedural/stub runtime и не делаем обязательным новый avatar rig pack.
- [ ] Зафиксировать, какие улучшения обязательны для всех аватаров, а какие включаются только для `near avatars` или при включенном `avatarLegIkEnabled`.
- [ ] Зафиксировать минимальный Phase 3 remote contract: какие поля уже берём из Phase 2 `pose/root/locomotion` path, какие поля обязательны для gait classification, и какой backward-compatible fallback применяется при их отсутствии.
- [ ] Зафиксировать deterministic fallback: при выключенном `avatarLegIkEnabled` или runtime incompatibility система возвращается к текущему stable body/pose path без поломки room flow.

### 2. Доработать locomotion classifier и transition layer

- [ ] Расширить `apps/runtime-web/src/avatar/avatar-locomotion.ts`, чтобы state machine различала `idle`, forward walk, strafe, backpedal и turn-in-place устойчиво на порогах и коротких сменах направления.
- [ ] Добавить hysteresis/smoothing на входах `moveX`, `moveZ`, `turnRate`, чтобы убрать дрожание между соседними состояниями.
- [ ] Ввести отдельные transition rules для `start`, `stop`, `sharp turn`, `strafe-to-forward`, `forward-to-backpedal` без резких визуальных скачков.
- [ ] Подготовить одинаковый classifier output для self/runtime path и remote/runtime path, чтобы не возникало разных gait decisions для одного и того же движения.

### 3. Усилить animation/gait layer

- [ ] Расширить `apps/runtime-web/src/avatar/avatar-animation.ts` от простого clip selection к blend-aware locomotion pose layer с предсказуемыми весами для `walk`, `strafe`, `backpedal`, `turn`.
- [ ] Добавить turn-in-place pose/animation behavior без "телепорта ног" и без деградации stationary head/hands tracking.
- [ ] Ввести smoothing на transitions между клипами/позами, чтобы при быстрых сменах направления не было stop-motion корпуса.
- [ ] Разделить near/far visual quality: near path получает более выраженный gait/body pass, far path - упрощенный locomotion mode с теми же состояниями, но меньшей стоимостью.

### 4. Добавить pelvis/spine refinement

- [ ] Добавить отдельный `pelvis offset`/weight shift layer, который реагирует на скорость и направление движения, а не только на head/hands solve.
- [ ] Добавить `spine lean` для `walk/strafe/backpedal` и отдельный `turn body twist` для поворота на месте.
- [ ] Зафиксировать profile-specific clamps для desktop/mobile/VR, чтобы улучшение корпуса не ломало текущие input profiles.
- [ ] Обеспечить, что при частичном XR input fallback или неполном remote pose runtime avatar остаётся в правдоподобном upper-body/body state, а не уходит в невалидную позу.

### 5. Ввести anti-foot-skating и near-avatar foot planting

- [ ] Ввести измеряемую `skating metric`, чтобы проверять, когда нижняя часть тела начинает визуально скользить относительно root motion.
- [ ] Реализовать базовое `lock/unlock` правило для anti-foot-skating на базе root speed, gait phase и transition state.
- [ ] Добавить упрощённый `near-avatar foot planting v1` без scene-aware collision/terrain logic и без тяжёлой физики.
- [ ] Ограничить foot planting по distance/LOD/profile, чтобы far avatars и слабые профили не тратили лишний budget.
- [ ] Зафиксировать fallback для случаев, где planting не может быть надежно вычислен из текущего pose/runtime input.

### 6. Подключить улучшения к remote runtime

- [ ] Расширить `apps/runtime-web/src/avatar/remote-avatar-runtime.ts`, чтобы remote avatars использовали тот же locomotion/body contract, а не только head/hands/root interpolation.
- [ ] Восстанавливать locomotion/body state для remote участника из уже доступного root/locomotion stream без требования нового transport payload, если текущих данных достаточно.
- [ ] Если текущего remote payload недостаточно для устойчивого gait classification, минимально расширить contract Phase 2 совместимым способом и документировать backward-compatible fallback.
- [ ] Сохранить существующую деградацию к coarse/stub presence path, если remote natural locomotion path временно невалиден или feature flag выключен.

### 7. Добавить rollout hooks и diagnostics

- [ ] Использовать существующий флаг `avatarLegIkEnabled` как главный rollout switch для Phase 3, не вводя лишний новый флаг без необходимости.
- [ ] Зафиксировать rollout order: сначала classifier/transition layer, потом self apply, потом remote apply, потом `near-avatar` refinement под `avatarLegIkEnabled`.
- [ ] Расширить avatar diagnostics/debug state метриками locomotion state, transition source, gait phase, anti-skating correction и near/far quality mode.
- [ ] Подготовить debug-friendly traces для сравнения `self` и `remote` поведения на одной и той же траектории.
- [ ] Явно описать, какие признаки считаются сигналом для rollback на staging.

### 8. Документация и завершение фазы

- [ ] Обновить `docs/2026-04-01-noah-avatar-system-tz-roadmap.md` и связанный план, чтобы границы Phase 3 были синхронизированы с фактической реализацией.
- [ ] Зафиксировать артефакты выхода фазы: `natural locomotion v1`, regression traces, staging evidence и список ограничений для Phase 4+.
- [ ] Отдельно задокументировать, что Phase 3 не требует нового avatar asset pack, но оставляет точку расширения для richer rigs в будущем.

## Затронутые файлы/модули (если известно)

- `docs/2026-04-01-noah-avatar-system-tz-roadmap.md`
- `docs/plans/2026-04-04-phase-3-avatar-natural-locomotion-v1.md`
- `packages/runtime-core/src/flags.ts`
- `apps/api/src/index.ts`
- `apps/runtime-web/src/index.ts`
- `apps/runtime-web/src/avatar/avatar-runtime.ts`
- `apps/runtime-web/src/avatar/avatar-controller.ts`
- `apps/runtime-web/src/avatar/avatar-animation.ts`
- `apps/runtime-web/src/avatar/avatar-locomotion.ts`
- `apps/runtime-web/src/avatar/avatar-ik.ts`
- `apps/runtime-web/src/avatar/remote-avatar-runtime.ts`
- `apps/runtime-web/src/avatar/avatar-instance.ts`
- `apps/runtime-web/src/avatar/avatar-debug.ts`
- `apps/runtime-web/src/avatar/*.test.ts`
- `apps/runtime-web/src/room-state-client.ts` (только если для remote gait contract понадобится минимальное расширение payload)
- `apps/room-state/src/index.ts` (только если понадобится совместимое расширение relay contract)

## CI / verification gates

- [ ] Blocking local gate: `pnpm lint`, `pnpm typecheck`, `pnpm build`, `pnpm test`.
- [ ] Blocking avatar regression gate: `record/replay` traces и deterministic bot paths для locomotion/body contract.
- [ ] Blocking published gate: `pnpm test:e2e:staging` после выката на staging.
- [ ] Manual evidence остаётся обязательной, но не заменяет automated gates для self/remote locomotion regressions.

## Тест-план

- **Unit**
- [ ] Тесты на пороги и hysteresis locomotion state machine: `idle/walk/strafe/backpedal/turn`.
- [ ] Тесты на clip/blend weights и smoothing переходов.
- [ ] Тесты на pelvis/spine refinement без выхода в невалидные позы.
- [ ] Тесты на anti-foot-skating correction и bounded behavior при резком stop/start.
- [ ] Тесты на `near/far` quality selection и fallback при выключенном `avatarLegIkEnabled`.
- [ ] Тесты на performance guardrails: включение Phase 3 не ломает boot/runtime path и не включает `near-avatar` refinement вне условий `LOD`.

- **Integration**
- [ ] `record/replay` трасс движения для self и remote с проверкой одинакового locomotion classification contract.
- [ ] Deterministic bot paths: `straight walk`, `circle`, `strafe`, `backpedal`, `stop`, `turn-in-place`.
- [ ] Проверка, что remote runtime не расходится с self runtime по состояниям на одних и тех же входных траекториях.
- [ ] Проверка graceful fallback при неполном remote pose stream, jitter и кратковременном отсутствии апдейтов.
- [ ] Проверка совместимости heavy scene rooms, чтобы новый body pass не ломал существующий room/avatar boot flow.
- [ ] Проверка, что far-avatar degrade сохраняет ту же state machine и меняет только визуальную детализацию/стоимость.

- **E2E / smoke**
- [ ] Локально прогнать обязательный набор: `pnpm lint`, `pnpm typecheck`, `pnpm build`, `pnpm test`, `pnpm test:e2e`.
- [ ] Добавить или расширить e2e-сценарии с минимум 2 клиентами, где проверяются locomotion transitions и отсутствие грубых remote regressions.
- [ ] После внедрения выкатить Phase 3 на staging и прогнать `pnpm test:e2e:staging`.
- [ ] На staging проверить не только `demo-room`, но и тяжелые scene rooms, где уже были чувствительные avatar regressions, включая `Hall`.

- **Manual**
- [ ] Два desktop клиента: walk/strafe/backpedal/stop/turn и визуальная оценка naturalness у self и remote.
- [ ] Desktop + Quest/WebXR: проверить, что turn-in-place, корпус и near-avatar interaction выглядят правдоподобно и не ломают VR tracking.
- [ ] Проверить near-avatar social distance сценарии: стоя рядом, обход, поворот на месте, короткие шаги, резкая остановка.
- [ ] Проверить far avatars: деградация должна быть мягкой, без резких артефактов и без скачков state.

- **Негативные кейсы**
- [ ] `avatarLegIkEnabled=false`: runtime возвращается к текущему stable locomotion/body behavior без поломки avatars.
- [ ] Неполный remote pose/runtime input: remote avatar не уходит в сломанную позу и деградирует предсказуемо.
- [ ] Резкая смена направлений около порогов: state machine не дребезжит между `walk/strafe/backpedal/idle`.
- [ ] Быстрый `turnRate` без движения: срабатывает turn-in-place, а не ложный walk/strafe.
- [ ] Heavy room / low-quality profile: near-avatar refinement отключается по правилам LOD, room flow не ломается.
- [ ] Scene-aware/terrain-aware planting отсутствует: упрощённый `foot planting v1` не должен пытаться решать неподдерживаемую геометрию сцены и уходить в нестабильное поведение.

## Риски и откаты (roll-back)

- Риск: Phase 3 разрастется в новый avatar asset/rig проект вместо эволюции текущего runtime.
  - Откат: держать фазу на текущем procedural/stub pipeline и выносить richer avatar assets в отдельную фазу.
- Риск: `self` и `remote` получат разную locomotion logic и начнут визуально расходиться.
  - Откат: использовать один classifier/transition contract и проверять его через `record/replay` на общих трассах.
- Риск: foot planting и anti-skating дадут слишком высокую стоимость.
  - Откат: ограничить эти улучшения `near avatars` и `LOD`-условиями, оставив far path упрощенным.
- Риск: улучшение корпуса сломает текущие VR/mobile профили.
  - Откат: держать profile-specific clamps/fallbacks и возможность полного выключения через `avatarLegIkEnabled`.
- Риск: rollout вызовет регрессии на staging в тяжелых сценах.
  - Откат: выключить `avatarLegIkEnabled`, вернуть Phase 2 stable path и откатить staging deployment до предыдущего SHA.
