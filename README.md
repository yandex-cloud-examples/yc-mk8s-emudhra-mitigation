# eMudhra CA Mitigation for Yandex Managed Kubernetes

DaemonSet отключает доверие к скомпрометированным цепочкам eMudhra на всех
Linux worker-нодах Yandex Managed Kubernetes, включая вновь добавленные ноды.
Обновление ОС и установка пакетов не требуются.

## Описание проблемы

Митигация удаляет из системного хранилища доверенных CA два корневых сертификата:

- emSign ECC Root CA - G3.
- AAA Certificate Services.

Основание: [рекомендация Ubuntu](https://ubuntu.com/blog/emudhra-ca-compromise-fixes-available)
и [USN-8944-1](https://ubuntu.com/security/notices/USN-8944-1).

## Что делает фикс

DaemonSet автоматически на каждой worker-ноде:

1. Удаляет указанные сертификаты из активного хранилища CA, сохраняя остальные сертификаты.
2. Перезапускает работающие `containerd` и `dockerd`, чтобы они использовали обновлённое хранилище.
3. Проверяет результат и восстановление сервисов.
4. Повторяет проверку и применение **раз в час**. Если изменений нет, сервисы повторно не перезапускаются.

Фикс действует на системное хранилище ноды. **Образы контейнеров и собственные
хранилища сертификатов приложений необходимо обновить отдельно.**

Обновление пакета CA или ручная пересборка хранилища могут вернуть удалённые
сертификаты. В этом случае митигация повторно удалит их в следующем часовом цикле;
до этого доверие к ним может восстановиться.

## Быстрый старт

### 1. Скачать манифест

```bash
wget https://raw.githubusercontent.com/yandex-cloud-examples/yc-mk8s-emudhra-mitigation/main/emudhra-mitigation-daemonset.yaml
```

Или клонировать репозиторий:

```bash
git clone https://github.com/yandex-cloud-examples/yc-mk8s-emudhra-mitigation.git
cd yc-mk8s-emudhra-mitigation
```

### 2. Применить фикс

Нужны права на создание ConfigMap и DaemonSet в `kube-system` с доступом к
процессам и хранилищу CA нод. Сначала проверьте фикс на тестовом кластере.
Первое применение выполняется на всех подходящих нодах; перезапуск runtime
может кратковременно прервать операции управления контейнерами.

```bash
kubectl config current-context
kubectl apply -f emudhra-mitigation-daemonset.yaml
```

### 3. Проверить статус

```bash
kubectl -n kube-system rollout status daemonset/emudhra-ca-mitigation --timeout=10m
kubectl -n kube-system get daemonset emudhra-ca-mitigation
kubectl -n kube-system get pods -l app=emudhra-ca-mitigation -o wide
kubectl get nodes
```

В DaemonSet значения `DESIRED`, `CURRENT`, `READY` и `AVAILABLE` должны совпадать
с числом ожидаемых Linux-нод. Ноды должны находиться в состоянии `Ready`.

### 4. Посмотреть логи

```bash
kubectl -n kube-system logs -l app=emudhra-ca-mitigation --prefix --tail=50
```

В логах каждой ноды должно появиться сообщение:

```text
PASS: affected roots absent; runtimes running with verified CA store
```

## Повторное применение

Чтобы применить фикс немедленно, не дожидаясь часового цикла:

```bash
kubectl -n kube-system rollout restart daemonset/emudhra-ca-mitigation
kubectl -n kube-system rollout status daemonset/emudhra-ca-mitigation --timeout=10m
```

## Удаление фикса

После обновления нод на исправленный базовый образ удалите временную митигацию:

```bash
kubectl delete -f emudhra-mitigation-daemonset.yaml
```

**Удаление DaemonSet не возвращает доверие к удалённым сертификатам**, но останавливает
периодическую проверку. Пересборка хранилища из неисправленных исходников может
снова добавить эти сертификаты.

## Технические сведения

- **Образ:** предустановленный `cr.yandex/yc/mk8s-openssl:stable`, `imagePullPolicy: Never`.
  Если образ отсутствует на ноде, pod не запустится с ошибкой `ErrImageNeverPull`.
- **Разрешения:** `hostPID`, запись в хранилище CA и каталог состояния; `privileged: false`,
  Linux capabilities удалены, AppArmor отключён для отправки сигналов host-сервисам.
- **Целостность:** скрипт проверяется по SHA-256 перед выполнением.
- **Ресурсы:** requests — 10m CPU / 32Mi RAM; limits — 500m CPU / 128Mi RAM.
- **Namespace:** `kube-system`.

## Совместимость

Предназначено для стандартных worker-нод Yandex Managed Kubernetes.
Проверено на текущих поддерживаемых версиях >= **1.32.1, Ubuntu **22.04.5**:
удаление CA, рестарты containerd и dockerd, сохранность работающего контейнера
и запуск новых контейнеров после применения.

## Лицензия

Apache License 2.0. См. [LICENSE](LICENSE).
