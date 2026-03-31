# План: Phase 5 — staging CD на VM через registry images и compose pull

## Цель

Перевести staging deploy `noah` с workspace/build-based path на reproducible CD path из registry: staging VM должна получать конкретные Docker images по immutable SHA tags и обновляться через `docker compose pull && docker compose up -d`, а GitHub Actions должен уметь запускать этот rollout по SSH и проверять базовый post-deploy smoke.

## Не-цель

- Не запускать обязательный `pnpm test:e2e:staging` автоматически после deploy; это следующая фаза.
- Не делать production deploy.
- Не менять бизнес-логику runtime/control-plane/storage, если это не требуется для registry-based deploy.
- Не добавлять сложный orchestrator, Terraform-managed deploy pipeline или multi-env promotion flow.
- Не менять staging topology: остается single-VM compose stack.

## Предпосылки и ограничения

- Phase 1-4 уже завершены: Dockerfiles есть, compose staging работает, storage abstraction работает, GitHub Actions публикует образы в `Yandex Container Registry`.
- Live registry contract уже существует:
  - `cr.yandex/crp9cm29k6p76hqo8lti/noah-api`
  - `cr.yandex/crp9cm29k6p76hqo8lti/noah-room-state`
- Current staging VM — `noah-stage-compose-v11` (`89.169.161.91`), и сейчас она все еще может собирать образы локально из checkout на VM.
- Goal этой фазы по roadmap: staging VM должна перейти на registry-based deploy path, а не продолжать `docker compose build` как основной путь.
- В этой фазе post-deploy verification ограничивается health/shell/control-plane smoke; full staging e2e gate остается на Phase 6.
- Deploy source of truth должен быть только immutable SHA image tag; alias tags не используются как deploy input.

## Подход

Сделать staging compose config image-driven: `api` и `room-state` тянутся по registry image refs, а GitHub Actions deploy workflow по SSH обновляет tag в env/config на VM, делает `docker compose pull` и `docker compose up -d`. Для этой фазы нужно выбрать один простой deploy contract: либо единый `IMAGE_TAG` для `api` и `room-state`, либо два отдельных SHA-параметра; реализация не должна разрастаться дальше этого. После Phase 5 `docker compose build` на staging остается только manual аварийным fallback, а не штатным deploy path. После rollout workflow проверяет `/health`, room shell и `control-plane` reachability. Rollback делается тем же способом, но с предыдущим SHA.

## Задачи

### 1. Зафиксировать deploy contract

- [ ] Зафиксировать, какие image refs использует staging VM для `api` и `room-state`.
- [ ] Зафиксировать способ передачи deploy SHA на VM: общий `IMAGE_TAG` или отдельные `API_IMAGE_TAG` / `ROOM_STATE_IMAGE_TAG`.
- [ ] Зафиксировать, какие файлы на VM остаются стабильными (`compose`, `.env`, volumes), а что меняется при deploy.
- [ ] Зафиксировать rule: staging deploy всегда идет по immutable SHA tag из registry.

### 2. Подготовить compose stack к pull-based deploy

- [ ] Обновить `infra/docker/compose.staging.yml`, чтобы `api` и `room-state` использовали image refs из registry, а не `build` как основной staging path.
- [ ] Сохранить локальный compose developer flow, не ломая local `build` use case.
- [ ] Обновить `infra/docker/.env.staging.example` и remote `.env.staging` contract для registry image refs / image tag.
- [ ] Зафиксировать registry login path для staging VM.

### 3. Подготовить staging VM под registry deploy

- [ ] Добавить на staging VM стабильный docker login path для `cr.yandex`.
- [ ] Зафиксировать, как и где на VM хранится registry auth.
- [ ] Убедиться, что compose VM умеет `docker compose pull` без ручного вмешательства.
- [ ] Проверить, что persistent volumes `postgres`, `minio`, `caddy` не зависят от старого build-based path.

### 4. Добавить GitHub Actions deploy workflow

- [ ] Добавить workflow/job для staging deploy по SSH после успешного publish-ready path.
- [ ] Передавать на VM конкретный image SHA tag, а не branch alias.
- [ ] По SSH обновлять staging env/config и запускать `docker compose pull && docker compose up -d`.
- [ ] Ограничить deploy workflow понятными triggers: вручную и/или push в staging branch, без deploy из PR.

### 5. Добавить post-deploy smoke

- [ ] После deploy проверять `/health` на публичном staging URL.
- [ ] Проверять room shell reachability хотя бы для `demo-room`.
- [ ] Проверять `control-plane` reachability.
- [ ] Явно падать, если post-deploy smoke не прошел.

### 6. Подготовить rollback path

- [ ] Зафиксировать rollback как повторный deploy предыдущего SHA image tag.
- [ ] Убедиться, что rollback не требует rebuild на VM.
- [ ] Проверить rollback smoke: `/health`, `demo-room`, `control-plane`.
- [ ] Зафиксировать, где брать previous successful SHA для ручного/автоматического отката.
- [ ] Зафиксировать, откуда deploy workflow берет previous successful SHA для rollback: workflow input, artifact, environment file или manual parameter.

### 7. Обновить документацию

- [ ] Обновить `docs/deployment-yandex-cloud.md` под registry-based staging rollout.
- [ ] Обновить `README.md` с новым staging deploy contract.
- [ ] Обновить project notes/`AGENTS.md`, если появятся важные operational lessons.

## Затронутые файлы/модули

- `.github/workflows/**`
- `infra/docker/compose.staging.yml`
- `infra/docker/.env.staging.example`
- `infra/yandex/cloud-init/staging-compose.yaml`
- `infra/yandex/scripts/provision-staging-compose.sh`
- `README.md`
- `docs/deployment-yandex-cloud.md`
- `AGENTS.md`

## Тест-план

- **Deploy contract**
- [ ] Staging compose config использует registry images по SHA, а не build path.
- [ ] VM успешно логинится в `cr.yandex` и делает `docker compose pull`.

- **Post-deploy smoke**
- [ ] После deploy `https://89.169.161.91.sslip.io/health` отвечает `200`.
- [ ] После deploy `https://89.169.161.91.sslip.io/rooms/demo-room` отвечает `200`.
- [ ] После deploy `https://89.169.161.91.sslip.io/control-plane` отвечает `200`.

- **Rollback**
- [ ] Rollback на предыдущий SHA выполняется без rebuild на VM.
- [ ] После rollback smoke endpoints остаются зелеными.

- **Негативные кейсы**
- [ ] При неверном SHA tag deploy job падает понятной ошибкой на этапе pull.
- [ ] При отсутствии registry auth deploy job fail-fast падает понятной ошибкой.
- [ ] При неуспешном rollout не теряются volumes `postgres` и `minio`.
- [ ] Alias tags не используются как фактический deploy input.

## Риски и откаты (roll-back)

- Риск: staging VM останется частично зависимой от старого checkout/build path.
  - Откат: явно разделить local build path и staging image path, не смешивать их в одном compose contract без env-guard.
- Риск: docker login на VM окажется хрупким или недолговечным.
  - Откат: документировать и автоматизировать registry auth refresh, но не делать build fallback default path.
- Риск: deploy workflow сможет случайно выкатить alias tag вместо SHA.
  - Откат: принимать на deploy только immutable SHA ref и валидировать формат тега.
- Риск: rollback останется ручным и непроверенным.
  - Откат: проверить хотя бы один реальный rollback smoke в этой фазе до закрытия плана.

## Definition of done для Phase 5

- [ ] Staging VM разворачивает `api` и `room-state` из registry images по SHA.
- [ ] Основной staging deploy path больше не требует `docker compose build` на VM.
- [ ] GitHub Actions умеет запускать staging deploy по SSH.
- [ ] После deploy автоматически проходит базовый post-deploy smoke: `/health`, `demo-room`, `control-plane`.
- [ ] Rollback на previous SHA документирован и проверен smoke-проверкой.
