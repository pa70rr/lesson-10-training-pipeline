# Automated training pipeline: GitLab CI і AWS Step Functions

## 1. High-level workflow automated training pipeline

```
GitLab event / manual trigger
↓
GitLab CI pipeline
↓
AWS Step Functions execution
↓
Data validation
↓
Training job
↓
Model evaluation
↓
Logging metrics and artifacts
↓
Model Registry update
↓
Notification to team
```

- Що запускає pipeline: подія в GitLab (push або merge у main, зміна model config, schedule) або manual trigger.
- Роль GitLab CI: стартує процес з Git-контексту, передає commit SHA, branch і source="gitlab-ci" та запускає Step Functions execution.
- Роль AWS Step Functions: керує послідовністю кроків від data validation до notification, показує статус кожного кроку, виконує retry і timeout, зупиняє workflow при помилці.
- Де фіксуються метрики: у MLflow разом із параметрами й артефактами training run.
- Де зберігається модель-кандидат: у Model Registry зі статусом candidate.
- Хто отримує повідомлення: команда в Slack або email - AI/ML Engineer, MLOps Engineer і AI PM. При помилці повідомлення адресоване owner-у етапу, на якому вона сталася.

## 2. Контрольні точки pipeline

| Етап pipeline | Що перевіряє команда | Хто owner | Що є успіхом | Що робити при помилці |
|---|---|---|---|---|
| Data validation | Наявність даних, схема, обов’язкові поля | Data Engineer | Дані пройшли перевірки | Зупинити pipeline, повідомити owner-а |
| Training job | Запуск тренування з правильною версією коду й даних | AI/ML Engineer | Training job завершився | Перевірити логи, повторити або створити incident |
| Model evaluation | Метрики моделі й порівняння з baseline | AI/ML Engineer + AI PM | Метрики вище threshold | Не просувати модель |
| Logging | Метрики, параметри й артефакти training run записані в MLflow | MLOps Engineer | Run і артефакти доступні в MLflow | Не реєструвати модель, перевірити logging step |
| Model Registry | Статус моделі-кандидата | MLOps Engineer | Модель зареєстрована як candidate | Перевірити registry/logging |
| Notification | Команда отримала зрозумілий статус | AI PM / MLOps Engineer | Є повідомлення з результатом | Додати alert або runbook |

## 3. Quality gates для нової моделі

| Quality gate | Приклад правила | Хто перевіряє | Рішення |
|---|---|---|---|
| Data quality | Немає критичних schema errors | Data Engineer | Продовжити або зупинити pipeline |
| Model metric | F1-score не нижче production baseline | AI/ML Engineer | Продовжити або rejected |
| Business metric | Немає погіршення на ключовому сегменті | AI PM | Погодити або повернути на аналіз |
| Approval | Модель має статус approved | AI PM / Tech Lead | Дозволити staging або production |

- ML-метрики: F1-score, precision і recall. Головна метрика погоджується до запуску pipeline.
- Порівняння: з поточною production-моделлю, а якщо її ще немає - з baseline.
- Мінімальний threshold: F1-score не нижче production baseline. Значення фіксується до запуску pipeline.
- Manual approval: потрібне перед переходом у staging і production.
- Хто ухвалює рішення: AI PM разом із Tech Lead.
- Якщо модель технічно навчилась, але не пройшла quality gates: вона отримує статус rejected у Model Registry, далі не просувається, у production залишається поточна модель, команда аналізує причину.

Технічний success pipeline означає лише, що всі кроки виконались без помилок. Готовність моделі до production визначають quality gates і approval.

## 4. Правила запуску training pipeline

| Trigger | Коли доречний | Перевага | Ризик | Рекомендоване середовище |
|---|---|---|---|---|
| Push у main | Навчальний або dev-сценарій | Простий запуск | Може створювати зайві training runs | dev |
| Manual trigger | Коли потрібен контроль | Менше випадкових запусків | Залежить від дисципліни команди | staging / prod |
| Schedule | Регулярний retraining | Передбачуваність | Може тренувати без реальної потреби | staging / prod |
| Drift signal | Реакція на деградацію | Запуск за реальною потребою | Потрібен зрілий monitoring | prod |

Правила запуску погоджуються заздалегідь. Частий автоматичний запуск створює зайві training runs і витрати, а надто ручний процес сповільнює retraining. Для prod кожен запуск має бути пов'язаний з manual trigger, розкладом або сигналом drift, і результат не потрапляє в production без approval.

## 5. Risk register

| Ризик | Ймовірність | Вплив | Як зменшити ризик | Owner |
|---|---|---|---|---|
| GitLab CI має надмірні AWS-права | Medium | High | Least privilege, OIDC, окремі ролі для dev/staging/prod | DevOps / MLOps |
| Модель не проходить quality gates | Medium | High | Порівняння з baseline, thresholds, manual approval | AI/ML Engineer + AI PM |
| Training запускається занадто часто | High | Medium | Manual trigger, schedule, rules by changed files | AI PM |
| Pipeline запускається без достатніх перевірок даних | Medium | High | Обов'язковий data quality gate перед training job | Data Engineer |
| У GitLab зберігаються постійні AWS keys | Medium | High | Перехід на OIDC, до переходу - masking і rotation ключів | DevOps / MLOps |
| Немає відповідального за помилки pipeline | Medium | High | Визначити owner-а кожного етапу й escalation path | AI PM |
| Модель автоматично просувається в production без approval | Low | High | Розділити training, registration, approval і production deployment | Tech Lead |

## 6. Acceptance criteria для Jira-задачі

Задача: Інтегрувати GitLab CI з AWS Step Functions для automated training pipeline.

- GitLab CI pipeline запускає AWS Step Functions workflow.
- У Step Functions input передаються commit SHA, branch і source="gitlab-ci".
- Step Functions execution видно в AWS Console.
- Pipeline має окремий data validation step перед training job.
- Training results логуються в MLflow або іншій системі tracking.
- Метрики й артефакти training run доступні для перевірки.
- Нова модель отримує статус у Model Registry.
- Команда отримує notification про success або failure.
- Доступ GitLab CI до AWS налаштований без надмірних прав.
- Quality gates перевіряються перед просуванням моделі.
- Нова модель не переходить у production без approval.

## 7. Notification messages

Success:

```
✅ Training pipeline completed successfully

Model: churn_prediction_v3
Source: GitLab CI
Commit: a1b2c3d
Step Functions execution: MLOpsPipeline / execution-123
Metrics: F1-score = 0.83, Recall = 0.79
Status: candidate registered
Details: MLflow run / Model Registry
Next action: Tech Lead reviews quality gates before release decision
Owner: Tech Lead
```

Failure:

```
❌ Training pipeline failed

Model: churn_prediction_v3
Source: GitLab CI
Commit: a1b2c3d
Failed step: DataValidation
Reason: Missing required column customer_age
Details: Step Functions execution logs / GitLab job logs
Next action: Data Engineer checks dataset and restarts pipeline after fix
Owner: Data Engineer
```

## 8. Короткий висновок

Automated training pipeline робить тренування моделі повторюваним і прозорим процесом замість ручного запуску notebook або script. GitLab CI запускає процес і пов'язує кожен training run з конкретним commit і branch. AWS Step Functions керує кроками від data validation до notification і показує, де процес завершився або впав. Технічний success pipeline означає лише, що всі кроки виконались, але не те, що нова модель краща за поточну. Перед production-релізом потрібні пройдені quality gates, порівняння з production-моделлю або baseline, approval від AI PM і Tech Lead та план rollback.
