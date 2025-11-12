# Стратегия тестирования — кратко и по делу

Применимые виды тестирования:

* Модульное (unit) — разработчики, покрытие критических библиотек и утилит.
* Интеграционное — взаимодействие CI → реестр образов → Helm/манифесты → Kubernetes API.
* Системное — end-to-end пайплайны от commit до рабочего Pod.
* Регрессионное — автоматизированные тесты пайплайнов и манифестов.
* Нефункциональное: нагрузочное, стресс, стабильности, безопасность (SAST/DAST), юзабилити, совместимость.
* Контроль конфигураций и инфраструктуры (infrastructure testing).

Инструменты:

* GitLab CI/CD — встроенные пайплайны, self-hosted опция, интеграция с реестром.
* Docker / Buildkit / Kaniko — сборка образов в CI.
* Helm / Kustomize — декларативный деплой в Kubernetes.
* Prometheus + Grafana — метрики, SLO/alerting.
* Loki / ELK — логирование для расследований.
* Trivy/Snyk/Clair — сканирование уязвимостей образов.
* JMeter / k6 — нагрузочное тестирование.
* Terraform / Ansible / Kubespray — инфраструктурный тестинг и репродуцируемость.

Роли и ресурсы:

* Тест-менеджер — стратегия, метрики, приемка.
* QA-инженеры (2) — функциональные и регрессионные тесты.
* SRE / DevOps (2) — тестовые окружения, мониторинг, откат.
* Разработчики — unit/integration тесты, исправление дефектов.
* Инфраструктурный инженер — настройка тестовых кластов и CI-раннеров.
  Ресурсы: тестовый Kubernetes-кластер (минимум 3 ноды для HA-симуляции), GitLab self-hosted CI, приватный реестр (Harbor/Registry), тестовая база данных, средства генерации трафика.

Критерии завершения тестирования:

* Функциональное покрытие >= 95% для критических сценариев.
* Нет открытых критических дефектов (Severity = Critical).
* Все автоматические пайплайны проходят в тестовом окружении не менее 3 подряд прогонов.
* Нефункциональные: latency p95 < целевого значения; throughput соответствует SLA; устойчивость при 2x ожидаемой нагрузки.
* Документированы и воспроизведены все найденные дефекты и тестовые наборы в системе трекинга.

Метрики качества тестирования:

* Coverage требований (Traceability %).
* Pass rate тест-кейсов (функц. / нефункц.).
* Mean time to detect (MTTD) дефекта.
* Mean time to fix (MTTR).
* Количество регрессий на выпуск.
* Время выполнения полного regression suite.

Процесс регистрации дефектов:

* Инструмент: GitLab Issues.
* Обязательные поля: шаги воспроизведения, среда (k8s версиия, образ), логи, severity, воспроизводимость.
* SLA на обработку: Critical — 24 ч, High — 72 ч, Medium — 7 дн.
* Схема triage: ежедневные стендапы для Critical/High.

# Функциональные тест-кейсы (18). Шаблон: ID | Title | Priority | Preconditions | Steps | Expected result | Req

1. TC-AUTH-001 | Успешная авторизация в веб-интерфейсе GitLab | Critical
   Pre: Создан user; доступ к UI.
   Steps: 1) Открыть /signin 2) Ввести валидные creds 3) Submit
   Expected: Вход успешен, пользователь видит dashboard.
   Req: AUTH-01

2. TC-AUTH-002 | Отказ при неверном пароле | High
   Pre: User существует.
   Steps: Ввести верный логин и неверный пароль.
   Expected: Отказ, сообщение об ошибке, блокировки нет.
   Req: AUTH-01

3. TC-CI-001 | Запуск пайплайна при push в ветку main | Critical
   Pre: Репозиторий с .gitlab-ci.yml.
   Steps: Сделать push в main.
   Expected: Пайплайн запускается, стадии build → test → deploy выполняются.
   Req: CI-01

4. TC-BUILD-001 | Успешная сборка docker-образа | Critical
   Pre: Dockerfile корректен, runner настроен.
   Steps: Запустить stage build.
   Expected: Образ собран и запушен в реестр, хэш совпадает с артефактом.
   Req: BUILD-01

5. TC-SCAN-001 | Сканирование образа на уязвимости обнаруживает критическую уязвимость | High
   Pre: Образ содержит уязвимость (тестовый образ).
   Steps: Запустить security scan stage.
   Expected: Сканы показывают уязвимость, пайплайн помечает результат как failed/warning по policy.
   Req: SEC-01

6. TC-DEPLOY-001 | Деплой в namespace staging | Critical
   Pre: Helm chart/manifest доступны.
   Steps: Выполнить stage deploy в staging.
   Expected: Deployment создан, Pods в Ready state, Service доступен.
   Req: DEP-01

7. TC-ROLLBACK-001 | Автоматический rollback при failed readiness | High
   Pre: Деплой настроен с readiness/liveness.
   Steps: Внедрить баг в образ, деплой.
   Expected: Health check не проходит, система выполняет rollback на предыдущую версию.
   Req: DEP-02

8. TC-SMOKE-001 | Smoke проверка после деплоя | High
   Pre: Деплой завершён.
   Steps: Выполнить набор smoke-запросов (health, basic API).
   Expected: Все ответы 200, ответ укладывается в SLA latency.
   Req: QA-01

9. TC-SERVICE-MESH-001 | Взаимодействие между сервисами через internal service | High
   Pre: Два микросервиса развернуты.
   Steps: Сделать запрос к сервису A, который вызывает B.
   Expected: Корректный ответ, трассировка видна в логах/trace.
   Req: INT-01

10. TC-SECRET-001 | Доступ к секретам через K8s Secret | Critical
    Pre: Secret создан.
    Steps: Деплой приложение, которое читает секрет из env.
    Expected: Приложение успешно читает значение, секрет не раскрыт в логе.
    Req: SEC-02

11. TC-NAMESPACES-001 | Изоляция между namespace (RBAC) | High
    Pre: Два namespace, роль ограничена.
    Steps: Попытаться получить ресурс из другого namespace с помощью serviceaccount.
    Expected: Доступ запрещён (403).
    Req: AUTH-02

12. TC-CANARY-001 | Canary deployment с частичным трафиком | Medium
    Pre: Canary strategy настроена.
    Steps: Выполнить canary deploy 10% трафика.
    Expected: 10% запросов идут к новой ревизии, метрики показывают распределение.
    Req: DEP-03

13. TC-LOGGING-001 | Логи приложений попадают в централизованную систему | Medium
    Pre: Promtail/agent настроен.
    Steps: Выполнить action в приложении, найти запись в Loki/ELK.
    Expected: Лог найден с корректными метаданными.
    Req: MON-01

14. TC-METRICS-001 | Экспорт пользовательских метрик в Prometheus | Medium
    Pre: Экспортер настроен.
    Steps: Сгенерировать нагрузку, проверить p99/p95 метрики.
    Expected: Метрики видны в Prometheus, Grafana графики обновляются.
    Req: MON-02

15. TC-HELM-001 | Применение values override | Medium
    Pre: Chart поддерживает values.
    Steps: Применить helm upgrade --values custom.yaml.
    Expected: Настройки применены, приложение работает с новыми параметрами.
    Req: OPS-01

16. TC-DB-CONN-001 | Подключение приложения к тестовой БД | Critical
    Pre: БД доступна, секреты заданы.
    Steps: Выполнить запрос записи и чтения.
    Expected: Операции успешны, откат транзакции работает.
    Req: INT-02

17. TC-CLEANUP-001 | Удаление ресурсов при destroy pipeline | Low
    Pre: Есть test namespace.
    Steps: Запустить cleanup stage.
    Expected: Ресурсы удалены, namespace очищен.
    Req: OPS-02

18. TC-AUDIT-001 | Аудит действий пользователей (who did what) | High
    Pre: Audit logging включён.
    Steps: Выполнить изменение в конфигурации, посмотреть audit logs.
    Expected: Запись содержит user, timestamp, action, resource.
    Req: SEC-03

# Нефункциональные тест-кейсы (6)

NF-LOAD-001 | Нагрузочное тестирование endpoint-а API | Critical
Pre: Test environment with scale.
Steps: Смоделировать ожидаемую и 2x ожидаемой нагрузку 30 минут.
Expected: p95 latency < SLA; error rate < 1%; нет падений кластера.

NF-STRESS-001 | Стрессовое тестирование | High
Pre: Система под мониторингом.
Steps: Увеличить нагрузку до отказа, затем снизить.
Expected: Система дегрейдит корректно, данные не теряются, восстановление происходит автоматически.

NF-STABILITY-001 | Тест на стабильность (soak) | High
Pre: Приложение под нагрузкой low-level.
Steps: 24-72 часа при обычной нагрузке.
Expected: Нет утечек памяти, ошибка < порога, средняя CPU нормальна.

NF-SEC-001 | Аудит безопасности (SAST + DAST + конфигурации) | Critical
Pre: Код и образы доступны для анализа.
Steps: Запустить Trivy/Snyk + ZAP/OWASP DAST.
Expected: Нет критических уязвимостей; для найденных — план mitigation.

NF-USAB-001 | Юзабилити интерфейса CI/CD для оператора | Medium
Pre: UI доступен.
Steps: Оператор выполняет набор задач: запустить пайплайн, инпут переменных, просмотреть логи.
Expected: Время выполнения задач укладывается в целевые значения; ошибки навигации < threshold.

NF-COMPAT-001 | Совместимость с Kubernetes версией (upgrade test) | High
Pre: Тестовый кластер доступен.
Steps: Обновить k8s minor version, прогнать deploys и smoke.
Expected: Система работает, манифесты совместимы, инструкции обновлены.

# Приоритеты и группировка

Группы: Авторизация/Аутентификация, CI/CD pipeline (build/test/deploy), Orchestration (K8s/Helm), Observability (metrics/logs/alerts), Security (secrets/audit/scans), DB/Integrations.

# Матрица трассируемости (приведу компактно)

Определим требования (обобщённо):

* R-AUTH: Аутентификация и RBAC
* R-CI: Автоматизация CI/CD
* R-BUILD: Сборка и реестр образов
* R-DEP: Деплой и откат
* R-SEC: Безопасность, сканирование, секреты
* R-MON: Мониторинг и логирование
* R-PERF: Производительность и стабильность
* R-OPS: Операционные сценарии (cleanup, upgrade)

Таблица соответствий (Тест → Требование):

* TC-AUTH-001, TC-AUTH-002, TC-NAMESPACES-001 → R-AUTH
* TC-CI-001, TC-BUILD-001, TC-HELM-001 → R-CI, R-BUILD
* TC-DEPLOY-001, TC-ROLLBACK-001, TC-CANARY-001 → R-DEP
* TC-SCAN-001, TC-SECRET-001, TC-AUDIT-001 → R-SEC
* TC-LOGGING-001, TC-METRICS-001 → R-MON
* TC-DB-CONN-001, TC-SERVICE-MESH-001 → R-OPS, R-INT
* NF-LOAD-001, NF-STRESS-001, NF-STABILITY-001 → R-PERF

Выявление требований без покрытия:

* Если в SRS есть требования про multi-region DR, backup retention и т.д. — добавить отдельные NF/OPS тесты.

# Целевые показатели покрытия и качества

* Покрытие требований: критические — 100%, высокие — >= 95%, остальные >= 80%.
* Unit-тесты: coverage модулей критической логики >= 80%.
* Regression pass rate: >= 98% при стабильной ветке.
* SLA по uptime тестовой среды: 99.5% в период тестирования.
* Время выполнения полного regression suite: целевое < 2 часа (при параллелизации).

# План тестирования (последовательность, сроки, ответственные)

Фазы:

1. Подготовка среды (2–4 дня) — SRE/DevOps

   * Поднять тестовый кластер, GitLab, runner, реестр, monitoring.
2. Unit & Integration (параллельно) (1–2 недели) — Разработчики + QA

   * Unit тесты, mock-интеграции.
3. Build & Security (2–3 дня) — QA + DevOps

   * Проверка сборки, сканирование образов.
4. System E2E (3–7 дней) — QA

   * Полные пайплайны, деплой, smoke.
5. Нефункциональные тесты (нагрузка/стабильность) (1–2 недели) — QA + SRE
6. UAT / OAT (3–5 дней) — Заказчик/операторы
7. Final regression и sign-off (2–3 дня) — Тест-менеджер

Ответственные:

* QA lead — координация сценариев.
* DevOps — поддержка среды и автоматизации.
* Devs — фиксы и unit tests.
* Product owner — UAT sign-off.

Процедура дефект-менеджмента:

* Создать issue → triage → assign → исправление → code review → пересборка → regression run → close.
* При регрессии: rollback и hotfix pipeline.
