# 🌡️ Zabbix Advanced CPU Temperature (Python/Sysfs)

![Zabbix 7.0+](https://img.shields.io/badge/Zabbix-7.0%2B-blue) ![Python 3](https://img.shields.io/badge/Python-3.x-yellow) ![Method Sysfs](https://img.shields.io/badge/Method-Direct_Sysfs-green) ![License](https://img.shields.io/badge/License-MIT-grey)

English documentation: [README.md](./README.md).

Продвинутое решение для мониторинга температуры процессора в Zabbix.

В отличие от стандартных решений, этот метод **не использует** парсинг вывода команды `sensors` (lm-sensors). Вместо этого используется Python-скрипт для прямого чтения значений из интерфейса ядра Linux (`/sys/class/hwmon`), что обеспечивает высокую скорость работы, отсутствие накладных расходов на запуск оболочки (shell) и независимость от версий утилит.

### 🚀 Возможности
* **Auto-Discovery (LLD):** Автоматически находит все ядра CPU и физические пакеты.
* **Smart Detection:** Скрипт сам определяет нужный драйвер (`coretemp` для Intel, `k10temp`/`zenpower` для AMD).
* **Метрики:** Температура каждого ядра + Агрегированные данные (Min / Max / Avg по всем ядрам).
* **Графики:** Автоматическое создание графиков для каждого ядра и сводного графика.
* **Zabbix 7.4 Ready:** Шаблон полностью совместим с новыми версиями Zabbix (UUIDv4 compliant).

---

## 🧠 Логика триггеров (Best Practices)

Используется стратегия «Воронки» (Funnel Strategy). Это позволяет фильтровать кратковременные скачки температуры (Turbo Boost / компиляция) и реагировать только на реальные проблемы с охлаждением.

| Температура | Задержка | Уровень | Описание проблемы |
| :--- | :--- | :--- | :--- |
| **> 70°C** | 30 мин | `Info` | Хроническая высокая нагрузка. Не критично, но стоит обратить внимание. |
| **> 80°C** | 10 мин | `Warning` | Неэффективное охлаждение. Температура не падает длительное время. |
| **> 90°C** | 3 мин | `Average` | **Перегрев.** Охлаждение не справляется с теплоотводом. Риск троттлинга. |
| **> 95°C** | 1 мин | `High` | Критическая зона. Близко к T-Junction max. |
| **> 100°C** | 0 мин | `Disaster` | Аварийная ситуация. Риск отключения сервера. |

---

## 🛠️ Быстрая установка (One-Click)

Вы можете установить скрипт и настроить агента одной командой:

```bash
curl -s https://raw.githubusercontent.com/didimozg/Zabbix-Temperature-CPU-Linux-Phyton/refs/heads/main/install.sh | sudo bash
```

## 🛠️ Ручная установка:

### 1. Python Скрипт
Скрипт занимается обнаружением ядер и чтением файлов sysfs.

1. Создайте папку для скриптов (если её нет):
   ```bash
   sudo mkdir -p /etc/zabbix/scripts

2. Скопируйте файл `zbx_py_cputemp.py` из этого репозитория в `/etc/zabbix/scripts/`.
3. Дайте права на выполнение:
```bash
sudo chmod +x /etc/zabbix/scripts/zbx_py_cputemp.py

```



### 2. Настройка Zabbix Agent

Нам нужно добавить всего одну строку `UserParameter`. Весь интеллект находится внутри Python-скрипта.

1. Создайте файл конфига `/etc/zabbix/zabbix_agent2.d/python_cputemp.conf`:
```ini
UserParameter=py.cputemp[*],/etc/zabbix/scripts/zbx_py_cputemp.py $1 $2

```


2. Перезапустите агент:
```bash
# Для Zabbix Agent 2
sudo systemctl restart zabbix-agent2

# Или для обычного агента
sudo systemctl restart zabbix-agent

```



### 3. Импорт шаблона

1. Скачайте файл `template_cputemp.yaml`.
2. В веб-интерфейсе Zabbix перейдите: **Data collection** → **Templates** → **Import**.
3. Загрузите файл и примените шаблон к нужному хосту.

---

## ❓ Troubleshooting

Проверить работу можно прямо из консоли сервера:

```bash
# 1. Проверка обнаружения (должен вернуться JSON)
zabbix_agent2 -t py.cputemp[discovery]

# 2. Проверка получения средней температуры
zabbix_agent2 -t py.cputemp[avg]

```

Если вы получаете ошибку прав доступа, убедитесь, что пользователь `zabbix` имеет доступ к чтению `/sys/class/hwmon/*` (обычно это доступно по умолчанию во всех дистрибутивах).

---

## Структура файлов

* `zbx_py_cputemp.py` — Основной скрипт коллектора.
* `template_cputemp.yaml` — Шаблон Zabbix (Triggers, Items, Graphs).
* `README_RU.md` — Инструкция.
