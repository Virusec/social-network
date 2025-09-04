1. Регистрация/вход пользователей.

Client App - API Gateway: запрос /auth/register или /auth/login. Сразу проверяет rate limit (Redis инкремент с TTL)  
API Gateway - Auth Service: пересылает запрос  
Auth Service - MongoDB: users:
- register: проверка уникальности (email/username), хеш пароля, запись профиля
- login: проверка пары логин/пароль

Auth Service: генерирует JWT (+при необходимости refresh-token), задаёт стандартные клеймы (userId, exp).  
Auth Service - Observability: метрика «успешный логин/регистрация».  
Auth Service - API Gateway - Client App: возвращает JWT.  
Все последующие запросы идут с Authorization: Bearer + JWT, API Gateway валидирует подпись/exp и пускает дальше.  

2. Публикация постов, комментирование.
3. Подписки/друзья.
4. Лента новостей.
5. Лайки, репосты.
