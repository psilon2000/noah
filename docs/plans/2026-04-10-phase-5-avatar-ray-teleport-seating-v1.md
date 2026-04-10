# План: Phase 5 - avatar ray, floor teleport и anchor seating `v1`

## Цель

Довести avatar subsystem до следующего продуктового шага после Phase 4: добавить единый interaction ray для desktop/web и WebXR, чтобы пользователь мог телепортироваться по полу и садиться на `seat anchors`, а состояние сидения было authoritative и одинаковым для всех участников комнаты.

## Не-цель

- Не делать generic object interaction, grab/use/manipulation и другие interaction systems beyond ray teleport + seating.
- Не делать сложную анимацию посадки, body IK, procedural alignment ног или сложный chair-fit solver.
- Не делать отдельный special-case только для fallback room; все сцены считаются равноправными и работают через единый anchor contract.
- Не делать видимый другим участникам луч; interaction ray виден только автору луча.
- Не превращать фазу в customization, LOD hardening или новый transport beyond того, что нужно для authoritative seating.

## Предпосылки и ограничения

- В roadmap `docs/2026-04-01-noah-avatar-system-tz-roadmap.md` Phase 5 уже закреплена как `Seating и scene integration`.
- В runtime уже есть avatar foundation, reliable avatar state, remote pose sync и lipsync, но нет production seating path и нет scene schema для seat anchors.
- В shared avatar contract уже есть `seated` и `seatId` в `apps/runtime-web/src/avatar/avatar-types.ts`; это даёт точку входа для reliable multiplayer seating без нового отдельного avatar identity contract.
- `apps/runtime-web/src/scene-bundle.ts` уже парсит versioned scene manifest; туда реалистично добавить anchors, не ломая текущий bundle path.
- `apps/room-state/src/state.ts` пока знает только про participants и не хранит authoritative seat occupancy; это нужно добавить явно.
- Пользователь зафиксировал main-path interaction contract:
- луч включается по явному действию;
- в VR луч вызывается жестом правого стика вверх;
- в web/desktop луч идёт от курсора;
- при указании на пол выполняется teleport;
- при указании на seat anchor выполняется sit;
- при повторном использовании луча из seated state допускается direct switch на другой seat или teleport без промежуточного отдельного stand шага;
- конфликт seat claim решается по правилу `first claim wins`.
- Если в сцене нет anchors, interaction ray всё равно может работать для floor teleport; sit UI/seat highlight просто не появляются.
- Если по ходу реализации scope окажется слишком тяжёлым, резать надо polish, а не один из двух main-path сценариев `teleport` или `seating`.

## Подход

Сделать одну простую interaction subsystem вокруг raycast target resolution. На клиенте луч определяет два типа целей: `floor-teleport target` и `seat anchor target`. Teleport по полу остаётся локальным avatar/root action, а посадка идёт через authoritative seat claim/release в `room-state`. Seat anchor описывает фиксированную позу посадки: позицию, yaw и высоту аватара относительно анчора. Runtime не пытается вычислять сложную посадку; он просто переводит local/remote avatar в `seated` mode и жёстко привязывает avatar root/body к anchor transform. Для всех сцен используется один schema contract, чтобы добавление seating было content/config задачей, а не кодовым special-case path.

## Definition of Done

Фаза завершена, если:

1. Пользователь может активировать локально видимый interaction ray в web и WebXR.
2. При валидном floor hit пользователь телепортируется без ломания existing room flow.
3. При валидном seat anchor hit пользователь садится только на свободный seat.
4. При конфликте двух пользователей за один seat сервер применяет `first claim wins`, второй пользователь остаётся в предыдущем состоянии.
5. Из seated state пользователь может либо пересесть на другой свободный seat, либо телепортироваться, при этом старый seat корректно освобождается.
6. Late join, reconnect и disconnect не оставляют zombie occupancy и показывают одинаковый seated state всем участникам.
7. Сцены без anchors продолжают работать без seating path и без runtime errors.
8. Локальные `pnpm build`, `pnpm test`, `pnpm test:e2e` и staging `pnpm test:e2e:staging` зелёные.

## Задачи (чек-лист)

### 1. Зафиксировать interaction и scene contracts Phase 5

- [ ] Описать единый runtime contract для interaction ray: activation, target resolution, highlight, confirm action и local-only visibility.
- [ ] Зафиксировать confirm contract для луча: отдельное confirm action для web и XR, не завязанное на сам факт показа луча.
- [ ] Добавить в scene bundle schema anchors contract с явным разделением минимум на `teleport floor` и `seat anchors`.
- [ ] Зафиксировать обязательные поля `seat anchor`: `id`, `position`, `yaw`, `seatHeight`, при необходимости `prompt`/`label`.
- [ ] Зафиксировать target zone contract для `seat anchors`: hit radius/shape и правило приоритета над floor hit.
- [ ] Зафиксировать, что в сценах без seat anchors seating path silently unavailable, но floor teleport остаётся рабочим.
- [ ] Зафиксировать, что в web луч идёт из камеры через курсорную точку экрана, а не через отдельный world-space pointer объект.
- [ ] Зафиксировать direct-switch semantics: из seated state новый valid ray action может сразу пересадить на другой seat или телепортировать пользователя.

### 2. Добавить authoritative seating state в room-state

- [ ] Расширить `apps/room-state` модель комнаты явным seat occupancy state, а не пытаться выводить занятость только из participant snapshots.
- [ ] Добавить сообщения/команды для `seat_claim` и `seat_release` с правилом `first claim wins`.
- [ ] Добавить authoritative server-side path для `seat_switch` как атомарной операции или эквивалентной server-controlled последовательности без client-side race.
- [ ] Сделать server-side reject path для конфликтного claim того же seat.
- [ ] Освобождать seat при disconnect/leave room и при успешном switch на другой seat.
- [ ] Гарантировать, что room snapshot и/или отдельные seat events восстанавливают корректную occupancy картину после reconnect/late join.

### 3. Довести client transport и avatar reliable wiring

- [ ] Расширить `apps/runtime-web/src/room-state-client.ts` поддержкой seat claim/release и authoritative seat updates.
- [ ] Не расширять unnecessarily `CompactPoseFrame`; seated state продолжает идти через reliable path.
- [ ] Убедиться, что local publisher (`avatar-publish.ts`) продолжает публиковать `seated` и `seatId` консистентно с server-ack состоянием, а не с optimistic-only client state.
- [ ] Зафиксировать seated publish contract: пока пользователь сидит, root/yaw публикуются от anchor pose, а не от обычного locomotion input.
- [ ] Добавить clear rollback-to-standing path при seat claim reject.
- [ ] Обновить avatar diagnostics/debug state так, чтобы были видны `rayTarget`, `seatClaimState`, `seatId`, `seated`, `seatRejectReason`.

### 4. Реализовать runtime interaction ray

- [ ] Вынести interaction ray logic из `main.ts` в отдельный runtime helper/module, чтобы не раздувать монолит.
- [ ] Для WebXR добавить activation path от правого стика вверх и корректный reset/debounce, чтобы луч не флапал каждый кадр.
- [ ] Для desktop/web добавить cursor-based ray targeting без ломания текущего look/mouse path.
- [ ] Сделать ray local-only visual с понятным highlight для пола и для seat anchors.
- [ ] Сделать приоритет target resolution: `seat anchor` выше floor teleport, если ray валидно попадает в anchor target zone.

### 5. Реализовать floor teleport main path

- [ ] Добавить raycast/resolution для допустимой teleport surface в текущей сцене.
- [ ] Зафиксировать минимальное правило teleport `v1`: teleport разрешён только на явно валидную floor surface/rule и не выполняется при invalid hit.
- [ ] При teleport из seated state сначала корректно освободить текущий seat на сервере, затем перевести avatar в standing position на target point.
- [ ] Обновить local avatar root/camera/player positioning так, чтобы teleport не ломал existing movement/presence path.
- [ ] Обновить remote observation path, чтобы другие участники видели уже результат teleport как normal avatar/root update без отдельной special animation.

### 6. Реализовать anchor seating main path

- [ ] Добавить `apps/runtime-web/src/avatar/avatar-seating.ts` с чистой логикой `claim/release/apply seated transform`.
- [ ] На успешный seat claim жёстко фиксировать local avatar относительно anchor position/yaw/seatHeight без сложной IK-посадки.
- [ ] При active seated state отключать обычный locomotion input и обычный floor teleport move до нового valid ray action.
- [ ] Разрешить direct switch: новый valid seat claim переводит пользователя на другой свободный seat без промежуточного ручного stand шага.
- [ ] Разрешить exit via teleport: teleport hit из seated state освобождает старый seat и переводит пользователя в standing mode.
- [ ] Обновить remote avatar application path, чтобы seated remote avatars визуально вставали в правильную anchor pose и одинаково выглядели у всех клиентов.

### 7. Интегрировать anchors в scene loading path

- [ ] Расширить `apps/runtime-web/src/scene-bundle.ts` parser и validator для новых anchor entries.
- [ ] Расширить scene loading/runtime state так, чтобы runtime имел нормализованный список anchors для активной сцены.
- [ ] Сделать safe path для сцен, где anchors отсутствуют или частично невалидны: seating отключается, room load не падает.
- [ ] Сделать safe fallback при потере/невалидности уже занятого anchor после загрузки сцены: принудительный safe stand без падения runtime.
- [ ] Подготовить минимум одну тестовую scene bundle c seat anchors для automated/staging acceptance.
- [ ] Зафиксировать content contract для будущего добавления anchors в любую сцену без новых кодовых веток.

### 8. Закрыть UX и крайние случаи

- [ ] Подсвечивать только текущую valid target zone, не показывая другим участникам ни луч, ни highlight.
- [ ] Не давать сесть на занятый seat; при server reject UI должен явно вернуть пользователя в previous valid state без зависания.
- [ ] Late join должен сразу видеть правильный seated state уже занятых мест.
- [ ] Reconnect seated user не должен дублировать occupancy или оставлять zombie seat lock.
- [ ] Disconnect seated user должен освобождать seat без ручной cleanup команды от клиента.
- [ ] Если сцена не содержит seat anchors, sit-specific UI не показывается, но floor teleport при наличии валидного пола остаётся доступным.

### 9. Довести фазу до shippable verification

- [ ] Добавить/обновить unit tests для scene anchor parsing, seat reducer/conflict logic и interaction target resolution.
- [ ] Добавить integration tests на `two users race for same seat`, `switch seat`, `teleport while seated`, `disconnect cleanup`, `late join while seats occupied`.
- [ ] Добавить локальный e2e на ray activation, floor teleport и sit/switch/exit базовые сценарии.
- [ ] Прогнать локально обязательный набор: `pnpm build`, `pnpm test`, `pnpm test:e2e`.
- [ ] После реализации выкатить изменения на staging обычным git-based flow и прогнать `pnpm test:e2e:staging`.
- [ ] Зафиксировать staging baseline для acceptance: минимум одна сцена с anchors и одна сцена без anchors.
- [ ] На staging вручную подтвердить desktop/web и XR main path: ray visible only locally, teleport работает, seat conflicts resolve correctly, reconnect/disconnect cleanup не ломает комнату.

## Затронутые файлы/модули (если известно)

- `docs/2026-04-01-noah-avatar-system-tz-roadmap.md`
- `docs/plans/2026-04-10-phase-5-avatar-ray-teleport-seating-v1.md`
- `apps/room-state/src/state.ts`
- `apps/room-state/src/index.ts`
- `apps/room-state/src/schema.ts`
- `apps/runtime-web/src/main.ts`
- `apps/runtime-web/src/room-state-client.ts`
- `apps/runtime-web/src/scene-bundle.ts`
- `apps/runtime-web/src/scene-loader.ts`
- `apps/runtime-web/src/avatar/avatar-seating.ts` (новый)
- `apps/runtime-web/src/avatar/avatar-runtime.ts`
- `apps/runtime-web/src/avatar/avatar-publish.ts`
- `apps/runtime-web/src/avatar/avatar-debug.ts`
- `apps/runtime-web/src/avatar/remote-avatar-runtime.ts`
- `apps/runtime-web/src/avatar/avatar-types.ts` (только если потребуется уточнить runtime-side seating contract)
- `packages/shared-types/src/avatar.ts` или соседние shared transport/state types, если seat messages выносятся в shared contract
- `tests/e2e/runtime.spec.ts`
- `tests/e2e/runtime-staging.spec.ts`
- `apps/runtime-web/public/assets/scenes/**/scene.json` для тестовой сцены с anchors

## Тест-план

### Unit

- [ ] parsing/validation scene anchors: валидный bundle проходит, битый anchor reject-ится без silent corruption.
- [ ] interaction target resolution: `seat anchor` имеет приоритет над floor target при пересечении валидной target zone.
- [ ] seat occupancy reducer: `claim`, `release`, `switch`, `disconnect cleanup` дают детерминированный room state.
- [ ] conflict rule: `first claim wins`, второй claimant получает reject без порчи occupancy.
- [ ] seated transform application: avatar получает ожидаемые `position/yaw/seatHeight` без накопления drift.

### Integration

- [ ] two users race for same seat.
- [ ] seated user switches to another free seat.
- [ ] seated user teleports to floor target и автоматически освобождает прошлый seat.
- [ ] reconnect while seated восстанавливает корректный occupied seat state без duplicate lock.
- [ ] late join получает уже занятые seats и правильно рисует seated remote avatars.
- [ ] occupied anchor becomes invalid/unavailable после scene/runtime reload и клиент уходит в safe standing state.
- [ ] scene without anchors не ломает runtime и не показывает sit path.

### E2E

- [ ] Локально: активация луча в web path, наведение на пол, teleport, наведение на seat anchor, посадка, switch seat.
- [ ] Локально: reject path при гонке за один seat между двумя клиентами.
- [ ] Локально: `pnpm build`.
- [ ] Локально: `pnpm test`.
- [ ] Локально: `pnpm test:e2e`.
- [ ] Staging: `pnpm test:e2e:staging` после обычной выкладки.

### Manual

- [ ] Desktop/web: курсорный луч, локальная подсветка, teleport по полу, sit/switch/exit.
- [ ] Desktop <-> desktop: два клиента видят одинаковый occupied/free state.
- [ ] Desktop <-> WebXR: VR луч от правого стика вверх, посадка и teleport не ломают avatar sync.
- [ ] Сцена без anchors: seating отсутствует, остальной room flow жив.

### Негативные кейсы

- [ ] Битый или неполный anchor config не валит room load; seating уходит в disabled/debuggable path.
- [ ] Seat claim reject не оставляет клиента в pseudo-seated состоянии.
- [ ] Disconnect seated пользователя не оставляет zombie occupancy.
- [ ] Быстрое многократное включение/выключение луча не создаёт stuck highlight/interaction mode.
- [ ] Частый repeat confirm не спамит `seat_claim` и не создаёт гонки из-за отсутствия debounce/cooldown.
- [ ] Teleport в invalid point не двигает пользователя и не портит authoritative state.
- [ ] Direct switch на уже занятый seat не снимает пользователя с текущего seat до получения валидного server ack.

## Риски и откаты (roll-back)

- Риск: фаза расползётся в generic interaction framework.
  - Откат: держать в обязательном scope только два target type `floor` и `seat`, без grab/use/object actions.
- Риск: seating снова разрастётся в сложную full-body animation/IK задачу.
  - Откат: seat anchor задаёт фиксированные `position/yaw/seatHeight`; animation polish не блокирует Phase 5.
- Риск: `main.ts` снова станет монолитом из ray/input/seating логики.
  - Откат: вынести interaction/seating helpers в `apps/runtime-web/src/avatar/*` или соседний runtime module и оставить `main.ts` wiring-слоем.
- Риск: optimistic client state начнёт расходиться с authoritative seat occupancy.
  - Откат: seat ownership подтверждается только через server ack/state update; reject path обязателен.
- Риск: direct seat switch сломает cleanup и оставит seat locks.
  - Откат: switch реализуется как атомарная server-side операция `claim new -> release old on success`, а не как два несвязанных client-side шага.
- Риск: anchor schema сломает старые scene bundles.
  - Откат: новые поля сделать optional; старые bundles продолжают грузиться без seating.
- Риск: teleport и seating вместе дадут слишком большой scope для одной фазы.
  - Откат: резать visual polish, не убирать ни floor teleport, ни authoritative seating из main path.
