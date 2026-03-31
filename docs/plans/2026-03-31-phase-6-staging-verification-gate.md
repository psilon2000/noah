# План: Phase 6 — staging verification как обязательный gate

## Цель

Сделать staging deploy для `noah` по-настоящему gate-driven: после rollout на staging GitHub Actions должен автоматически запускать `pnpm test:e2e:staging`, проверять обязательные staging flows и считать deploy успешным только если post-deploy smoke и staging verification зеленые; при fail workflow должен помечать deploy неуспешным и выполнять rollback на previous successful SHA.

## Не-цель

- Не делать production verification/deploy.
- Не расширять продуктовую логику runtime/control-plane/storage beyond того, что нужно для надежного staging gate.
- Не строить отдельную сложную orchestration систему из нескольких environments/promotions.
- Не заменять текущий registry-based deploy contract; Phase 6 работает поверх уже внедренной Phase 5 схемы.

## Предпосылки и ограничения

- Phase 5 завершена: staging deploy уже идет из registry images по immutable SHA, есть рабочий workflow `.github/workflows/staging-deploy.yml`, verified `workflow_dispatch` run и проверенный rollback по previous SHA.
- Есть существующий staging smoke workflow `.github/workflows/staging-smoke.yml` и staging Playwright suite `tests/e2e/runtime-staging.spec.ts`.
- Staging e2e уже ходят по public HTTPS URL и покрывают selector flow + весь восстановленный scene catalog.
- Текущий staging host: `https://89.169.161.91.sslip.io`.
- Для rollback workflow нужен previous successful SHA; его нужно хранить/обновлять предсказуемо и не путать с alias tags.
- Жесткий gate должен жить прямо в `Staging Deploy`, а не в side workflow, чтобы deploy без verification не считался успешным.

## Подход

Расширить `.github/workflows/staging-deploy.yml` так, чтобы после rollout и базового smoke он запускал staging Playwright suite на публичном HTTPS URL. Mandatory gate использует текущий стабилизированный staging suite и не должен автоматически расширяться flaky/manual-only checks без отдельного решения. Если smoke или `pnpm test:e2e:staging` падают, workflow выполняет rollback на previous successful SHA, повторно проверяет минимальный smoke и завершает job как failed. Источник previous successful SHA должен быть простым и устойчивым: сохраняемый deploy state на VM и/или workflow input/output, без дополнительной внешней БД.

В реализации нужно оставить mandatory gate минимально детерминированным: текущий staging suite плюс already-stabilized scene checks, без автоматического добавления дополнительных manual/test-only rooms.

## Задачи

### 1. Зафиксировать gate contract

- [ ] Зафиксировать список staging checks, которые обязательны всегда после deploy: `/health`, `demo-room`, `control-plane`, `pnpm test:e2e:staging`.
- [ ] Зафиксировать, какие staging tests остаются частью mandatory gate, а какие могут быть manual-only в будущем.
- [ ] Зафиксировать rollback trigger: любой fail post-deploy smoke или `pnpm test:e2e:staging`.
- [ ] Зафиксировать источник previous successful SHA для rollback.

### 2. Интегрировать staging e2e в deploy workflow

- [ ] Обновить `.github/workflows/staging-deploy.yml`, чтобы после rollout выполнялся Playwright-based `pnpm test:e2e:staging`.
- [ ] Передавать в workflow нужные env vars для staging suite (`BASE_URL`, scene room ids, admin token при необходимости).
- [ ] Убедиться, что staging suite использует public HTTPS path, а не direct IP fallback.
- [ ] Сделать verification steps blocking для успешного deploy status.

### 3. Упростить/встроить existing staging smoke path

- [ ] Решить, остается ли `.github/workflows/staging-smoke.yml` как manual utility workflow, или его логика полностью переезжает в `staging-deploy.yml`.
- [ ] Убрать дублирование wait/health logic между workflow'ами.
- [ ] Зафиксировать один source of truth для staging verification contract.

### 4. Реализовать rollback on failed verification

- [ ] Добавить в deploy workflow rollback step, который выполняется при fail verification.
- [ ] Rollback должен повторно rollout'ить previous successful SHA без rebuild на VM.
- [ ] После rollback выполнять минимальный smoke: `/health`, `demo-room`, `control-plane`.
- [ ] Даже после успешного rollback исходный failed deploy должен оставаться failed в GitHub Actions.

### 5. Зафиксировать successful deploy state

- [ ] Добавить простой механизм сохранения current successful SHA для следующего rollback.
- [ ] Зафиксировать, где хранится этот state: файл на VM, env file, workflow artifact или другой простой persistent path.
- [ ] Убедиться, что rollback не опирается на alias tags вроде `staging`.
- [ ] Явно записывать current successful SHA и rollback target в workflow logs/summary для отладки.

### 6. Обновить документацию

- [ ] Обновить `README.md` с новым gate-driven staging deploy contract.
- [ ] Обновить `docs/deployment-yandex-cloud.md` с описанием auto verification и rollback behavior.
- [ ] Обновить `AGENTS.md`, если появятся важные operational lessons по gate/rollback behavior.

## Затронутые файлы/модули

- `.github/workflows/staging-deploy.yml`
- `.github/workflows/staging-smoke.yml`
- `tests/e2e/runtime-staging.spec.ts`
- `infra/docker/rollout-staging-images.sh`
- `README.md`
- `docs/deployment-yandex-cloud.md`
- `AGENTS.md`

## Тест-план

- **Workflow path**
- [ ] `Staging Deploy` выполняет rollout, smoke и `pnpm test:e2e:staging` в одном workflow.
- [ ] Successful deploy остается green только если все checks прошли.

- **Verification gate**
- [ ] После deploy `https://89.169.161.91.sslip.io/health` отвечает `200`.
- [ ] После deploy `https://89.169.161.91.sslip.io/rooms/demo-room` отвечает `200`.
- [ ] После deploy `https://89.169.161.91.sslip.io/control-plane` отвечает `200`.
- [ ] `pnpm test:e2e:staging` проходит на public HTTPS base URL.

- **Rollback**
- [ ] Если verification intentionally fails, workflow выполняет rollback на previous successful SHA.
- [ ] После rollback минимальный smoke снова зеленый.
- [ ] Failed rollout остается failed в GitHub Actions history, even if rollback succeeded.

- **Негативные кейсы**
- [ ] При сломанном staging e2e gate deploy job fail-fast уходит в rollback.
- [ ] При отсутствии previous successful SHA workflow падает явной ошибкой rollback_precondition_failed.
- [ ] При неудачном rollback workflow дает явный failed status и не маскирует проблему.
- [ ] Rollback не удаляет `postgres`/`minio` volumes.

## Риски и откаты (roll-back)

- Риск: staging e2e сделают deploy слишком долгим и flaky.
  - Откат: разделять mandatory checks и heavier optional checks, но mandatory gate должен оставаться deterministic.
- Риск: rollback logic станет слишком сложной из-за хранения previous SHA.
  - Откат: использовать максимально простой persistent state, например файл на staging VM.
- Риск: flaky scene checks будут часто откатывать рабочий deploy.
  - Откат: держать в mandatory gate только уже стабилизированные staging tests; остальное оставлять manual-only или advisory.
- Риск: workflow станет трудно отлаживать после rollback.
  - Откат: всегда сохранять явные logs/status для failed deploy и отдельный лог rollback path.

## Definition of done для Phase 6

- [ ] `Staging Deploy` включает обязательный post-deploy gate с `pnpm test:e2e:staging`.
- [ ] Deploy считается успешным только если smoke и staging e2e зеленые.
- [ ] При fail verification workflow автоматически откатывает previous successful SHA.
- [ ] Rollback path проверен реальным failed rollout scenario.
- [ ] Gate/rollback contract задокументирован в проекте.
