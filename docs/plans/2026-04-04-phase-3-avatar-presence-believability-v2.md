# План: Phase 3 - believable avatar presence without legs `v2`

Статус: REFRAMED

## Цель

Сделать аватары воспринимаемо живыми и стабильными без обязательных ног, gait solver и procedural body motion. Главный результат фазы - не "походка", а комфортный avatar presence: без дёрганий, без сломанного VR, с предсказуемым self/remote поведением и хорошим качеством рук, головы и поворотов.

## Не-цель

- Не пытаться имитировать мета-аватар с ногами, если текущий продукт хорошо работает без ног.
- Не считать "скольжение без шагов" дефектом само по себе.
- Не добавлять fake torso lean, foot planting, anti-foot-skating и прочую synthetic body animation как обязательный продуктовый scope.
- Не вводить новый mesh/rig/avatar pack.
- Не трогать lipsync, seating, customization и rich hand-tracking transport.

## Предпосылки и ограничения

- Практический фидбек по первой версии Phase 3 показал, что procedural natural locomotion не дал заметной пользы в web и дал регрессии в VR.
- Хорошие avatar-системы могут выглядеть убедительно и без ног, если upper-body presence, head/hands visibility и remote sync работают стабильно.
- Phase 2 already solved the hard multiplayer base: room-state sync, reconnect, same-browser identity, heavy scene boot ordering, remote hand visibility.
- Значит следующая полезная фаза должна усиливать believability и stability, а не имитировать "ходьбу любой ценой".

## Подход

Сместить фокус с "body naturalness" на "presence believability". За продуктовый baseline принимаем no-leg avatar style. В фазе улучшаем только то, что реально влияет на качество восприятия: стабильность головы и рук, отсутствие регрессий в VR, плавность remote/self sync, предсказуемые повороты и переходы, качественная диагностика и staging regression gates. Всё, что относится к ногам, gait solver, foot planting и body sway, выводится в отдельный experimental/research path и не считается обязательным результатом продуктовой фазы.

## Definition of Done

Фаза завершена, если:

1. Self и remote avatars выглядят стабильно и не раздражают в desktop и VR сценариях.
2. В VR пользователь не видит собственную голову, руки отображаются корректно и не пропадают у remote observers.
3. В web и VR нет synthetic body jitter, лишних поворотов корпуса и других procedural артефактов.
4. Remote avatar sync, visibility и transitions не ломаются в `demo-room` и тяжёлых staging rooms.
5. Локальные `pnpm build`, `pnpm test`, `pnpm test:e2e` и staging `pnpm test:e2e:staging` зелёные.
6. Ручная проверка подтверждает, что новая версия не хуже Phase 2 по ощущению качества.

## Что уже стало ясно

- Первая формулировка Phase 3 как `legs/gait solver/body naturalness` была неверной продуктовой гипотезой.
- Метрика успеха здесь не "стало больше движения тела", а "аватар не бесит и не ломает immersion".
- Current natural-locomotion branch полезна как исследование и как набор regression tests, но не как обязательный продуктовый baseline.

## Experimental / research path

- Код вокруг `avatarLegIkEnabled`, `avatarik=1`, locomotion trace harness и forced staging checks остаётся как experimental path.
- Этот path можно использовать для будущих исследований richer avatars или upper-body polish.
- Но он не должен быть default product requirement, пока не появится явная пользовательская ценность и не исчезнут VR regressions.

## Задачи (чек-лист)

### 1. Зафиксировать новый product contract

- [ ] Переписать roadmap и Phase 3 формулировки с `legs/natural locomotion` на `believable avatar presence without legs`.
- [ ] Явно отделить product baseline от experimental locomotion branch.
- [ ] Зафиксировать, что smooth sliding без ног - допустимое поведение, если оно выглядит лучше и стабильнее.

Статус: выполнено в roadmap и текущем runtime contract; product baseline теперь рассматривается как smooth no-leg presence, а experimental path вынесен под override.

### 2. Закрыть обязательные product-quality кейсы

- [ ] Гарантировать скрытие локальной головы в VR.
- [ ] Гарантировать стабильную видимость remote VR рук в web.
- [ ] Гарантировать отсутствие body jitter / torso twist / strafe artifacts.
- [ ] Гарантировать, что remote/self transitions и visibility не ломают existing avatar sync path.

Статус: закрыто кодом и regression tests для self VR visibility, remote VR hand visibility и calmer no-leg upper-body baseline.

### 3. Усилить automated verification

- [ ] Держать staging e2e на legacy-safe path по умолчанию.
- [ ] Держать отдельные staging checks на experimental path под query override, чтобы исследования не ломали production baseline.
- [ ] Держать regression tests на локальную VR голову, remote VR руки и strafe stability.
- [ ] Держать heavy-room staging checks для `Hall` и других чувствительных комнат.

Статус: покрыто local/staging e2e и unit tests; baseline идёт по умолчанию, experimental `avatarik=1` проверяется отдельно.

### 4. Провести обзорный acceptance pass

- [ ] Desktop: базовое движение, повороты, two-client remote observation.
- [ ] VR: head hidden, hands stable, no jitter.
- [ ] Desktop <-> VR: remote visibility и общее ощущение качества.
- [ ] Подтвердить, что текущий baseline не хуже предыдущей стабильной версии.

## Затронутые файлы/модули

- `docs/2026-04-01-noah-avatar-system-tz-roadmap.md`
- `docs/plans/2026-04-04-phase-3-avatar-presence-believability-v2.md`
- `apps/runtime-web/src/avatar/avatar-controller.ts`
- `apps/runtime-web/src/avatar/remote-avatar-runtime.ts`
- `apps/runtime-web/src/avatar/avatar-debug.ts`
- `apps/runtime-web/src/avatar/avatar-runtime.ts`
- `apps/runtime-web/src/main.ts`
- `tests/e2e/runtime.spec.ts`
- `tests/e2e/runtime-staging.spec.ts`

## Тест-план

- **Unit**
- [ ] visibility/head-hidden invariants
- [ ] remote VR hand visibility invariants
- [ ] no-stray torso rotation for strafe / upper-body stability invariants

- **Integration**
- [ ] record/replay остаётся для experimental locomotion branch, но не является главным acceptance критерием product baseline
- [ ] multi-client self/remote presence checks

- **E2E**
- [ ] `pnpm build`
- [ ] `pnpm test`
- [ ] `pnpm test:e2e`
- [ ] `pnpm test:e2e:staging`
- [ ] отдельные staging checks на legacy path и experimental path

- **Manual**
- [ ] desktop/desktop
- [ ] desktop + Quest/WebXR
- [ ] оценка "не стало ли хуже"

## Риски и откаты

- Риск: снова начать оптимизировать абстрактную "естественность", а не реальное восприятие качества.
  - Откат: любой новый body-motion path держать только за флагом и принимать только после ручной оценки.
- Риск: experimental branch снова просочится в default behavior.
  - Откат: legacy-safe path остаётся default, исследования идут только через query/env override.
- Риск: roadmap опять закрепит спорную гипотезу как обязательную фазу.
  - Откат: описывать экспериментальные пути отдельно от product commitments.
