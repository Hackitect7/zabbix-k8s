# Zabbix в Kubernetes (alpine) — manifests + мини Helm-чарт + CI/CD в Docker Hub

**Namespace (Docker Hub):** `hackitect7`

Проект разворачивает три компонента Zabbix в Kubernetes из контейнеров на базе **alpine-latest** и предоставляет два способа установки: plain YAML (manifests) и минимальный Helm-чарт. Репозиторий снабжён GitHub Actions, который:

1. поднимает кластер `kind`,
2. деплоит Zabbix и ждёт готовности,
3. выполняет smoke‑test HTTP,
4. публикует Helm‑чарт (OCI) и образ с вложенным `tar.gz` (bundle) в Docker Hub.

---

## Состав

```text
.
├─ manifests/
│  ├─ 00-namespace.yaml
│  ├─ 10-secret-db.yaml
│  ├─ 20-postgres-sts.yaml          # postgres:16-alpine (PVC)
│  ├─ 30-zabbix-server-deploy.yaml  # zabbix/zabbix-server-pgsql:alpine-latest
│  ├─ 40-zabbix-web-deploy.yaml     # zabbix/zabbix-web-nginx-pgsql:alpine-latest
│  └─ 50-ingress.yaml               # Ingress (spec.ingressClassName)
├─ helm/
│  └─ zabbix-mini-chart/
│     ├─ Chart.yaml
│     ├─ values.yaml
│     └─ templates/*.yaml
└─ .github/workflows/ci.yml
   └─Dockerfile.bundle
```

**Bundle**: архив `zabbix-k8s-bundle.tar.gz` содержит каталог `manifests/` и `helm/zabbix-mini-chart/`.

---

## Предварительные требования

* Kubernetes-кластер (локально: Docker Desktop с включённым Kubernetes или `kind`).
* `kubectl` ≥ 1.26, `helm` ≥ 3.12, `docker`.

---

## Быстрый старт (локально)

### Вариант A — plain manifests

```bash
kubectl apply -f manifests/00-namespace.yaml
kubectl apply -f manifests/10-secret-db.yaml
kubectl apply -f manifests/20-postgres-sts.yaml
kubectl apply -f manifests/30-zabbix-server-deploy.yaml
kubectl apply -f manifests/40-zabbix-web-deploy.yaml
kubectl apply -f manifests/50-ingress.yaml

kubectl -n zabbix wait --for=condition=available deploy/zabbix-server --timeout=600s
kubectl -n zabbix wait --for=condition=available deploy/zabbix-web --timeout=600s

# Доступ к UI
kubectl -n zabbix port-forward svc/zabbix-web 8080:80
# http://localhost:8080 (Admin / zabbix)
```

### Вариант B — Helm-чарт (из Docker Hub, OCI)

```bash
helm registry login registry-1.docker.io -u hackitect7
helm install zabbix oci://registry-1.docker.io/hackitect7/zabbix-mini-chart \
  --version 0.1.0 \
  --namespace zabbix --create-namespace

kubectl -n zabbix wait --for=condition=available deploy/zabbix-web --timeout=600s
kubectl -n zabbix port-forward svc/zabbix-web 8080:80
```

> Если используете Ingress локально, поставьте контроллер `ingress-nginx` и используйте `spec.ingressClassName: nginx`. Для локального теста достаточно `port-forward`.

---

## Параметры (values)

`helm/zabbix-mini-chart/values.yaml` (важные фрагменты):

```yaml
namespace: zabbix
web:
  host: "zabbix.local"
  timezone: "Asia/Bishkek"
  ingressClassName: "nginx"
```

**Переменные окружения** в контейнерах Zabbix корректно названы:

* `DB_SERVER_HOST`, `DB_SERVER_PORT`, `DB_SERVER_DBNAME`
* `POSTGRES_USER`, `POSTGRES_PASSWORD`
* (web) `ZBX_SERVER_HOST`, `ZBX_SERVER_PORT`, `PHP_TZ`

**PostgreSQL**: `postgres:16-alpine` в StatefulSet + PVC (по умолчанию 10Gi, RWO).

---

## CI/CD (GitHub Actions)

### Job `test-deploy`

* Поднимает кластер `kind`.
* Применяет `manifests/*`.
* Ждёт readiness `zabbix-server` и `zabbix-web`.
* Выполняет smoke‑test: HTTP‑ответ от `zabbix-web` изнутри кластера с проверкой слова `Zabbix`.

### Job `publish`

* Публикует Helm-чарт в Docker Hub как **OCI-чарт**:

  * `oci://registry-1.docker.io/hackitect7/zabbix-mini-chart:0.1.0`
* Собирает образ с вложенным `zabbix-k8s-bundle.tar.gz` и пушит в Docker Hub:

  * `docker.io/hackitect7/zabbix-k8s-bundle:manifests-0.1.0`

> Публикация включена для `push`/`workflow_dispatch` и отключена для `pull_request`, чтобы секреты были доступны.

---

## Как получить bundle из образа

**Linux/macOS**

```bash
docker pull docker.io/hackitect7/zabbix-k8s-bundle:manifests-0.1.0
cid=$(docker create docker.io/hackitect7/zabbix-k8s-bundle:manifests-0.1.0)
docker cp "$cid:/bundle/zabbix-k8s-bundle.tar.gz" .
docker rm "$cid"

tar -tzf zabbix-k8s-bundle.tar.gz | head
```

**Windows PowerShell**

```powershell
docker pull docker.io/hackitect7/zabbix-k8s-bundle:manifests-0.1.0
$cid = docker create docker.io/hackitect7/zabbix-k8s-bundle:manifests-0.1.0
docker cp "$cid`:/bundle/zabbix-k8s-bundle.tar.gz" .
docker rm $cid

tar -tzf .\zabbix-k8s-bundle.tar.gz | Select-Object -First 20
```

---

## Проверка/приёмка

1. **Образы**:

```bash
kubectl -n zabbix get deploy zabbix-server -o=jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
kubectl -n zabbix get deploy zabbix-web    -o=jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
kubectl -n zabbix get sts zabbix-postgres  -o=jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
```

Ожидаемо: `zabbix/zabbix-server-pgsql:alpine-latest`, `zabbix/zabbix-web-nginx-pgsql:alpine-latest`, `postgres:16-alpine`.

2. **Готовность**:

```bash
kubectl -n zabbix wait --for=condition=available deploy/zabbix-server --timeout=600s
kubectl -n zabbix wait --for=condition=available deploy/zabbix-web    --timeout=600s
```

3. **UI**:

```bash
kubectl -n zabbix port-forward svc/zabbix-web 8080:80
# Открыть http://localhost:8080 — логин Admin / пароль zabbix (смените после входа)
```

4. **Smoke‑test (внутри кластера)**:

```bash
kubectl -n zabbix run curl --image=curlimages/curl:8.9.1 --restart=Never --rm -i -- \
  sh -c 'curl -s -o - http://zabbix-web.zabbix.svc.cluster.local/ | grep -qi "Zabbix"'
```

5. **Docker Hub**:

* чарт виден в `hackitect7/zabbix-mini-chart` с тегом `0.1.0`,
* образ `hackitect7/zabbix-k8s-bundle:manifests-0.1.0` присутствует и тянется.

---

## Типичные проблемы и решения

* **503 от Ingress**: нет эндпоинтов у `zabbix-web` → ждём readiness, проверяем `kubectl -n zabbix get endpoints zabbix-web`. Для локали чаще используйте `port-forward`.
* **`CrashLoopBackOff` у web/server и `password authentication failed` в Postgres**: проверьте имена env (`DB_SERVER_DBNAME`, `POSTGRES_USER/PASSWORD`), при необходимости удалите PVC БД для переинициализации (демо-среда).
* **Порт 8080 уже занят**: используйте `kubectl port-forward ... 18080:80` и открывайте `http://localhost:18080`.
* **401 при публикации**: проверьте namespace/секреты в GitHub, для ORAS — предварительно создайте репозиторий; в Варианте B (image) это не требуется.

---

## Рекомендации для продакшена

* Зафиксировать версии образов (`7.4-alpine-latest`, `16-alpine`) вместо скользящих `latest`.
* TLS (cert-manager), `NetworkPolicy`, `PodSecurity (PSA)`, секреты через SealedSecrets/SOPS.
* Бэкапы БД (CronJob + `pg_dump` в объектное хранилище).
* HA Zabbix Server + TimescaleDB для истории, PDB/PodAntiAffinity, HPA для `zabbix-web`.
* CI: chart-testing, релизы по тэгам, подпись OCI (cosign).
