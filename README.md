Задача:
 Написать скрипт на bash для мониторинга процесса test в среде
 linux. Скрипт должен отвечать следующим требованиям:
 1. Запускаться при запуске системы (предпочтительно написать юнит
 systemd в дополнение к скрипту)
 2. Отрабатывать каждую минуту
 3. Если процесс запущен, то стучаться(по https) на
 https://test.com/monitoring/test/api
 4. Если процесс был перезапущен, писать в лог /var/log/monitoring.log
 (если процесс не запущен, то ничего не делать)
 5. Если сервер мониторинга не доступен, так же писать в лог.
 # Вся установка производиться из фала Quick start, копированием двух скриптов
   Тестировалось в окружении rocky linux 9
 # Проверка задачи по шагам

# 1) Запускаться при запуске системы (юнит systemd)
 Проверяем, что таймер включён (значит, поднимется на старте системы) и привязан к юниту:
 включён ли таймер в автозагрузке
 
- systemctl is-enabled test-monitor.timer

 таймер загружен и активен с момента старта
 
systemctl status test-monitor.timer

 увидеть, какой юнит он триггерит
 
systemctl cat test-monitor.timer | sed -n '1,120p'

 Должно быть: Triggers: test-monitor.service и OnUnitActiveSec=60s

# 2) Отрабатывать каждую минуту
 в списке таймеров видно NEXT/LAST, каждая минута
 
systemctl list-timers | grep test-monitor

 убедиться, что интервал именно 60s
 
grep -E 'OnUnitActiveSec|OnBootSec' /etc/systemd/system/test-monitor.timer

# 3) Если процесс запущен — стучаться по HTTPS на https://test.com/monitoring/test/api

 (В примере у нас myapp.service как «процесс».)

 Убедиться, что целевой процесс запущен:
 
systemctl is-active myapp.service   # должно быть "active"

 Указать требуемый URL и запустить проверку:
 
 поставить нужный URL из ТЗ
 
sudo sed -i 's|^MON_URL=.*|MON_URL="https://test.com/monitoring/test/api"|' /etc/sysconfig/test-monitor

 очистим лог для наглядности
 
sudo truncate -s0 /var/log/monitoring.log

 разовый запуск мониторинга
 
sudo systemctl start test-monitor.service

 Что смотреть в логе (в зависимости от ответа сервера):
 
sudo tail -n 50 /var/log/monitoring.log

 Если сервер отвечает 2xx → лог может быть пуст (это норма: успех не логируем), либо появится RECOVERED (http=2xx) если до этого было «down»
 
 Если не-2xx → появится NON-2XX (http=...).
 
 Если сеть/SSL/DNS упали → UNREACHABLE (exit=...).
 
 (Альтернатива «доказать, что запрос ушёл» — временно поставить MON_URL="https://httpbin.org/status/204" и увидеть RECOVERED (http=204) при первом «подъёме».)

# 4) Если процесс был перезапущен — писать в лог /var/log/monitoring.log (если процесс не запущен — ничего не делать)
# A) Ловим перезапуск:
 перезапускаем целевой сервис
 
sudo systemctl restart myapp.service

 вручную запускаем монитор (или ждём минуту до тика таймера)
 
sudo systemctl start test-monitor.service

 проверяем лог: должна появиться строка "process restarted: ..."
 
sudo tail -n 50 /var/log/monitoring.log | grep 'process restarted' || true

# B) Проверяем «если процесс не запущен — ничего не делать»:
 останавливаем процесс
 
sudo systemctl stop myapp.service

 чистим лог для наглядности
 
sudo truncate -s0 /var/log/monitoring.log

 запускаем монитор
 
sudo systemctl start test-monitor.service

 лог должен остаться пустым (скрипт корректно "ничего не делает")
 
sudo tail -n 50 /var/log/monitoring.log

# 5) «Если сервер мониторинга недоступен — писать в лог»
 Имитируем недоступность (нерезолвимый домен):
 
sudo sed -i 's|^MON_URL=.*|MON_URL="https://nonexistent.example.invalid/api"|' /etc/sysconfig/test-monitor

sudo systemctl start test-monitor.service

sudo tail -n 50 /var/log/monitoring.log | grep 'UNREACHABLE' || true

Ожидаем запись вида:

monitoring server UNREACHABLE (exit=6|28|...) url=...

(6 — DNS, 28 — таймаут и т.п.)

Пара удобных «сбросов» между тестами

# вернуть "зелёный" URL (успех 204)

sudo sed -i 's|^MON_URL=.*|MON_URL="https://httpbin.org/status/204"|' /etc/sysconfig/test-monitor

# очистить лог

sudo truncate -s0 /var/log/monitoring.log




