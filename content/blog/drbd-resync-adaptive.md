---
title: "DRBD resync положил диски Jenkins на 8 часов"
date: 2026-05-06
description: "w_await на primary-ноде вырос до 103ms во время resync 1.1 TB раздела Jenkins. Переход с фиксированного resync-rate на адаптивный контроллер снизил пик до 5.7ms без остановки Jenkins."
tags: ["drbd", "jenkins", "storage", "disaster-recovery"]
---

Во время полной ресинхронизации DRBD-раздела Jenkins (1.1 TB) `w_await` на primary-ноде вырос до 103ms. Builds деградировали, агенты начали таймаутиться. Причина — фиксированный `resync-rate` без учёта нагрузки приложения. Решение заняло 15 минут.

## Контекст

Jenkins работает в active-standby DR: primary-нода держит раздел DRBD в состоянии `Primary/UpToDate`, secondary — `Secondary/UpToDate`. Оба узла на OpenStack Ubuntu 22.04, раздел `jenkins` занимает 1.1 TB из 1.3 TB блочного устройства.

После планового обслуживания secondary-нода вышла из синхронизации. DRBD начал полный resync.

## Проблема

Сразу после старта resync `iostat` на primary показал рабочую нагрузку:

```text
Device  r/s   w/s    wkB/s   w_await  %util
vdb     945  2270   12969     5.71    34.4
```

Через 10 минут под активной нагрузкой Jenkins:

```text
Device  r/s   w/s    wkB/s   w_await  %util
vdb     980    54    5613   103.00    76.0
```

`w_await` 103ms — это очередь на диске. Builds стали зависать, часть агентов ушла в JNLP reconnect loop.

Конфиг resync в момент инцидента:

```bash
drbdsetup disk-options jenkins --c-plan-ahead=0 --resync-rate=512000k
```

`c-plan-ahead=0` отключает адаптивный контроллер. DRBD читал с диска ~945 r/s для отправки на secondary и не снижал скорость, даже когда Jenkins активно писал.

## Решение

Переключил на adaptive resync прямо на ходу — без остановки Jenkins, без разрыва репликации:

```bash
drbdsetup disk-options jenkins \
  --c-plan-ahead=20 \
  --c-fill-target=50k \
  --c-min-rate=50000k \
  --c-max-rate=500000k
```

Что делает каждый параметр:

`c-plan-ahead=20` — горизонт планирования контроллера в единицах 0.1s, то есть 2 секунды. Документация рекомендует ≥ 5x RTT; при ping 0.23ms между нодами значение 20 избыточно, но безопасно.

`c-fill-target=50k` — сколько данных держать in-flight, в секторах (1 сектор = 512 байт, 50k ≈ 25 MB). Контроллер регулирует скорость чтения так, чтобы очередь не росла.

`c-min-rate=50000k` — нижняя граница скорости resync. Даже когда Jenkins пишет интенсивно, контроллер не опустит resync ниже ~50 MB/s.

`c-max-rate=500000k` — верхний предел. Когда диск свободен — resync берёт до ~500 MB/s.

## Результат

`iostat` через 2 минуты после смены параметров:

```text
Device  r/s   w/s    wkB/s   w_await  %util
vdb     870  2269   12965     5.71    34.4   # Jenkins активно пишет
vdb       0     0       0     0.00     0.0   # Jenkins idle
vdb       1     1       4     0.00     0.0   # Jenkins idle
```

`w_await` — максимум 5.7ms против 103ms до изменений. Builds перестали зависать.

Скорость resync упала до ~36 MB/s: DRBD отдаёт приоритет Jenkins и берёт остатки. Расчётное время до конца синхронизации — 8.5 часов. Jenkins работал без перебоев всё это время.

## Выводы

Фиксированный `resync-rate` с `c-plan-ahead=0` на production-ноде — ошибка конфигурации. Adaptive controller включён в DRBD 9 по умолчанию (`c-plan-ahead=20`), но явный `--c-plan-ahead=0` в `drbdsetup disk-options` его отключает.

Если меняешь параметры через `drbdsetup` — проверь, что `c-plan-ahead` не ноль. Параметры применяются без перезапуска ресурса и без прерывания репликации.

Следующий раз: сначала adaptive, потом смотреть, нужен ли fixed rate.

---

- [DRBD 9 man page: disk-options](https://manpages.debian.org/unstable/drbd-utils/drbd.conf-9.0.5.en.html)
- [LINBIT: Tuning the DRBD Resync Controller](https://kb.linbit.com/drbd/tuning-the-drbd-resync-controller/)
