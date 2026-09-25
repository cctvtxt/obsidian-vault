# Руководство: конфигурация k8s (k8s/)

## 1. За что отвечают конфиги `k8s/`

Структура — Kustomize: `base/` (общие манифесты) + `overlays/{dev,prod}`
(надстройки под окружение). Применяется через
`kubectl apply -k k8s/overlays/<env>`.

### k8s/base
- `namespace.yaml` — Namespace `backend` (изоляция ресурсов).
- `configmap.yaml` — `backend-config`: НЕсекретные env (`GRPC_HOST`/`PORT`,
  `*_SERVICE_URL`, `REDIS_*`, `POSTGRES_*`, `EXP_DAYS`, `JWT_ISSUER`,
  `RESET_INTERVAL_MINUTES`, `CORS_ORIGIN`). Читается всеми сервисами через
  `envFrom: configMapRef`.
- `secret.yaml` — `backend-secret` (`POSTGRES_USER`/`PASSWORD`, `NGROK_*`):
  gitignored, в VCS не коммитится (см. «Чего не хватает»).
- `deployment.yaml` — 4 Deployment (api-gateway, auth/user/token-service) +
  2 StatefulSet (postgres, redis). Для каждого: образ, `imagePullPolicy: Always`,
  gRPC-порты, `env` (`NODE_ENV=production`, `GRPC_PORT`, `DATABASE_URL`, собранный
  из `POSTGRES_*`), `readiness`/`livenessProbe` (grpc для сервисов, http `/healthz`
  для gateway), `envFrom` configMap+secret.
- `service.yaml` — headless ClusterIP (`clusterIP: None`) для gRPC-сервисов
  (нужно для обнаружения по имени в кластере) и обычные ClusterIP для
  gateway/postgres/redis.
- `ngrok.yaml` — Deployment туннеля ngrok → публичный URL на api-gateway
  (внешний доступ без Ingress/LB).

### k8s/overlays
- `prod`: `kustomization.yaml` наследует base + патчит `replicas` (stay 1) и
  `imagePullPolicy: IfNotPresent` (образы уже в minikube-cache, не тянуть каждый раз).
- `dev`: `kustomization.yaml` ссылается на `patchesStrategicMerge: patch-replicas.yaml`
  — но файла `patch-replicas.yaml` в `overlays/dev/` **нет** (есть только
  `patch-image-pull-policy.yaml`) ⇒ `kubectl apply -k k8s/overlays/dev` упадёт.
  Реальный дефект: патч-файл не создан (в base replicas уже =1, патч избыточен —
  строку `patchesStrategicMerge` можно убрать).

### Связь с репозиторием
- Makefile: `up-prod`/`restart-prod`/`pull-prod` поднимают minikube и
  `kubectl apply -k k8s/overlays/prod` + ждут rollout.
- CI `cd.yaml`: `docker buildx bake` → пуши → self-hosted раннер: minikube,
  `minikube image pull`, `kubectl apply -k k8s/overlays/prod`, rollout restart,
  проверка ngrok-туннеля по `/healthz`.

## 2. Различия запуска «k8s» и minikube

- **minikube** — локальный одноузловой кластер (VM/контейнер/Docker, driver).
  Для dev/CI, эмулирует API k8s локально. Образы: либо из локального docker
  (`minikube image load` / `eval $(minikube docker-env)`), либо pull в minikube.
  Хранилище — hostPath/emptyDir; сети — внутри узла; нет HA.
- **«настоящий» k8s** (managed EKS/GKE/AKS или self-hosted) — control plane +
  пул worker-узлов, отказоустойчивость, облачные LoadBalancer/Ingress,
  StorageClass, внешние Secret-стораджи.
- **В этом репозитории оба используют одни и те же манифесты** (kustomize).
  minikube выступает «локальным продом»: `*-prod` таргеты Makefile и CD-раннер
  реально крутят minikube, а не cloud-k8s. Разница проявляется в том, откуда
  тянутся образы (Docker Hub vs локальный minikube-cache) и в отсутствии
  облачных LB/Ingress (внешний доступ только через ngrok).

## 3. Чего не хватает / что ещё можно настроить

- **resource requests/limits** (CPU/mem) — сейчас нет ⇒ непредсказуемый
  шедулинг и риск OOM.
- **HPA / VPA** — автомасштабирование (сейчас replicas жёстко =1).
- **Ingress / Gateway API** — внешний доступ только через ngrok (туннель в
  обход k8s-сети); нет терминации TLS в кластере.
- **securityContext / PodSecurityContext** — `runAsNonRoot`, `readOnlyRootFilesystem`,
  drop capabilities — критично для прода.
- **NetworkPolicy** — сервисы общаются свободно; нет сегментации.
- **ResourceQuota / LimitRange** на namespace `backend`.
- **Probes для postgres/redis** (StatefulSet) — сейчас без readiness/liveness.
- **initContainer с `prisma migrate deploy`** вместо выполнения миграций в
  `entrypoint.sh` (декларативнее, раньше готовность).
- **Иммутабельные теги образов** — `:latest` мешает rollback/воспроизводимости
  (лучше tag/digest по версии).
- **Сторонние секреты** — `secret.yaml` gitignored и правится вручную; в проде
  нужны Sealed Secrets / External Secrets / Vault. Сейчас `apply` упадёт без
  локально созданного секрета.
- **PodDisruptionBudget, affinity/topologySpread** — для выкатов без простоя.
- **Логи/метрики/трейсы** — нет стека наблюдаемости.
- **TLS/HTTPS на gateway** (сейчас Fastify http, трафик шифрует ngrok).

## 4. Принципы настройки k8s

1. **Декларативность + GitOps**: манифесты в VCS, применение через
   `kubectl apply -k` / `helm upgrade`; окружения — через kustomize/helm, а не
   правкой вручную.
2. **ConfigMap vs Secret**: конфиг — в ConfigMap, секреты — в Secret; секреты
   никогда не коммитить (использовать внешние сторы).
3. **12-factor**: конфиг через env, один процесс/контейнер, образ
   неизменяем (всё через деплой, не in-place правки).
4. **Probes**: readiness (пускать трафик), liveness (перезапуск при зависании),
   startupProbe для медленного старта.
5. **Ресурсы обязательны**: requests/limits — основа шедулинга и стабильности.
6. **Idempotent rollouts**: rolling update + лёгкий откат (immutable tags).
7. **Least privilege**: securityContext, NetworkPolicy, RBAC.
8. **Состояние вне подов**: stateless-приложение в Deployment, состояние —
   StatefulSet + PVC или управляемая БД снаружи.
9. **Observability**: метрики/логи/трейсы из коробки.
10. **Namespaces** для изоляции окружений/команд.
