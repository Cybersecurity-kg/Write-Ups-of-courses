# Authentication Vulnerabilities – Write‑up по Hint‑решениям (Web Security Academy)

Каждая секция: описание, эксплуатация/PoC и защита.

---

## 1. Username enumeration via different responses (Lab #1)  
**Описание:** Различие текста или `Content-Length` при невалидном vs валидном логине позволяет определить существующие логины.  
**Эксплуатация:** Burp Intruder (Sniper или Cluster Bomb) или ffuf: перебор логинов с фиксированным паролем, фильтрация по длине или ответу. Валидный логин вернёт «Incorrect password», остальные — «Invalid username» :contentReference[oaicite:0]{index=0}  
**Защита:** Унификация ошибок и длины ответа, стабилизация тайминга, rate‑limit по логину/IP.

---

## 2. 2FA simple bypass (Lab #2)  
**Описание:** Возможен обход второго фактора: доступ к защищённой странице без ввода кода при прямом переходе на `/my-account` после логина.  
**Эксплуатация:** Залогиниться, получить код, заметить URL `/my-account`. Залогиниться жертве, затем вручную изменить URL на `/my-account` — позволяет обойти 2FA этап :contentReference[oaicite:1]{index=1}  
**Защита:** Проверка состояния 2FA на защищённых страницах, отказ от прямого доступа без прохождения MFA.

---

## 3. Password reset broken logic (Lab #3)  
**Описание:** Сброс пароля не проверяет токен или user‑binding; можно изменить пароль другого пользователя без token.  
**Эксплуатация:** Сгенерировать reset для `wiener`, перехватить запрос, поменять `username` на `carlos`, оставить токен пустым или удалить → пароль изменится и можно войти как `carlos` :contentReference[oaicite:2]{index=2}  
**Защита:** Валидировать токен, привязывать к user/session, проверять temp token, использовать одноразовые токены и проверку email.

---

## 4. Username enumeration via response timing (Lab #5)  
**Описание:** Тайминги ответов различны для существующих логинов: проверка пароля затратна только если логин валиден.  
**Эксплуатация:** Burp Intruder или Turbo: замер времени ответа по списку логинов → валидные заметно медленнее :contentReference[oaicite:3]{index=3}  
**Защита:** Тайм‑обфускация, искусственные задержки, rate‑limit, детект обхода через `X‑Forwarded‑For`.

---

## 5. Broken brute‑force protection, IP block (Lab #6)  
**Описание:** Система блокирует IP после нескольких неудачных попыток, но счётчик сбрасывается после успешного входа другого пользователя.  
**Эксплуатация:** Попытки ввести неправильный пароль → блокировка. Ввод валидной пары (wiener:peter) сбрасывает счётчик и позволяет продолжать перебор на `carlos` :contentReference[oaicite:4]{index=4}  
**Защита:** Корреляция ошибок по user и IP, блокировка без reset после чужой успешной попытки, rate‑limit.

---

## 6. Brute‑forcing a stay‑logged‑in cookie (Lab #9)  
**Описание:** Токен `remember‑me` закодирован как `base64(username:md5(pass))`; можно декодировать и брутфорсить.  
**Эксплуатация:** Перехват текущего токена, декодирование base64, подбор md5‑хеша → узнать пароль, либо brute‑force с форматом username:hash → сгенерировать токен и вставить его вручную → получить доступ :contentReference[oaicite:5]{index=5}  
**Защита:** Использовать подпись (HMAC), криптографически стойкие токены, привязку к устройству/IP, TTL и ревокацию.

---

## 7. Offline password cracking (Lab #10)  
**Описание:** Токен `stay‑logged‑in` содержит md5‑хеш пароля; можно получить токен offline и cracked pass.  
**Эксплуатация:** Утечка токена (через XSS) → декод base64 → извлечь md5‑хеш → crack с онлайн‑словарём (например, Crackstation) → получить пароль пользователя :contentReference[oaicite:6]{index=6}  
**Защита:** отказ от md5‑хеша, использовать bcrypt/Argon2 + соль, ограничить lifespan токенов.

---

## 8. Password reset poisoning via middleware (Lab #11)  
**Описание:** Уязвим в заголовке `X‑Forwarded‑Host`: можно изменить ссылку сброса в письме на домен атакующего.  
**Эксплуатация:** Перехват запроса reset, добавить `X‑Forwarded‑Host: attacker.com`, сменить `username` на `carlos` → ссылка в письме ведёт на сервер атаки, жертва нажимает → сброс пароля `carlos` → злоумышленник контролирует трафик :contentReference[oaicite:7]{index=7}  
**Защита:** Жёсткий базовый URL на сервере, игнорирование полей `X‑Forwarded-Host`, проверка домена.

---

## 9. Password brute‑force via password change (Lab #12)  
**Описание:** Изменение пароля позволяет брутфорсить чужой пароль через endpoint `change-password` без logout.  
**Эксплуатация:** Логин как `wiener`, отправить change-request с неправильным старым паролем, но разные новые пароли → различие в ответе (status или logout) указывает на валидный старый пароль для пользователя `carlos` при установленном `username=carlos` :contentReference[oaicite:8]{index=8}  
**Защита:** Проверка текущего пароля, logout сразу при неверном вводе, конфиденциальность ошибок, rate-limit.

---

## 10. Broken brute‑force protection: multiple credentials per request (Lab #13)  
**Описание:** API позволяет передавать список паролей в одном запросе и не соблюдает лимиты.  
**Эксплуатация:** Отправка `{ "username": "carlos", "password": ["123", "password", ...] }` → получение авторизационного cookie без отдельной попытки → вставка в браузер даёт доступ :contentReference[oaicite:9]{index=9}  
**Защита:** Блокировка множественных credentials, ограничение payload size, rate‑limit per attempt.

---

## 11. 2FA broken logic (Lab #8)  
**Описание:** Возможен обход MFA методом brute‑форса `verify` параметра или OTP-кода после установки `verify=carlos`.  
**Эксплуатация:** Перехват `GET /login2`, изменить `verify` на `carlos`. Затем brute flux OTP через Intruder → успешный вход и доступ к `/my-account` :contentReference[oaicite:10]{index=10}  
**Защита:** Привязка MFA-кода к user/session, rate-limit на OTP попытки, проверка прав пользователя перед генерацией кода.

---

