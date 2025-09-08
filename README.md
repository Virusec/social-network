Проектирование социальной сети  
Функциональные требования:
- Регистрация/вход пользователей.
- Публикация постов, комментирование.
- Подписки/друзья.
- Лента новостей.
- Лайки, репосты.

Нефункциональные требования:  
- Высокая доступность.
- Возможность горизонтального масштабирования.
- Использование NoSQL.
- Возможна eventual consistency (лайки могут приходить с задержкой).
- Хранение мультимедиа.
- Rate limiting для защиты от спама.




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


Подписка/дружба (Social Graph)

Клиент → Gateway: POST /follow/{userB} (с JWT). Rate limit проходит.

Gateway → Social Graph: команда follow(userA, userB).

Social Graph → Mongo (social_graph):

пишет ребро A → B (и опционально обратный индекс B ← A для быстрого выборки подписчиков).

Social Graph → Observability: метрика/лог.

Сообщение в Event Bus (опционально): FollowCreated (если нужны уведомления).

Ответ клиенту.

Публикация поста (Post) + раскладка в ленты (Feed) + мультимедиа (Media)
A. Загрузка мультимедиа

Клиент → Gateway → Media: инициация загрузки (multipart или pre-signed URL). Rate limit.

Media → Object Storage: кладёт файл(ы), может запустить асинхронную генерацию превью/thumbnail.

Media → Клиент: отдаёт ссылки на объекты (objectUrl) и/или CDN-URL (как только CDN прогреется).

B. Создание поста

Клиент → Gateway → Post: POST /posts (текст + ссылки на objectUrl).

Post → Mongo (posts): запись документа поста {postId, authorId, text, media[], createdAt…}.

Post → Event Bus: публикует PostCreated {postId, authorId, createdAt}.

Post → Observability: метрики/логи.

Post → Клиент: подтверждение.

C. Fan-out на запись (Feed)

Feed ← Event Bus: получает PostCreated.

Feed → Mongo (social_graph): читает список подписчиков автора батчами/пейджингом.

Feed → Mongo (timelines): для каждого подписчика делает запись «входящего элемента ленты»
{userId=subscriberId, postId, createdAt, rank…} (bulk-insert, шардирование по userId).

Feed → Observability: метрика «рассылок» (fan-out) и лаг потребителя.

Если подписчиков очень много, Feed шардируется, раскладывает пачками и может продолжать рассылку после того, как Post уже вернул успех — это и есть eventual consistency: у части подписчиков пост появится с небольшой задержкой.

Комментирование (Comment)

Клиент → Gateway → Comment: POST /posts/{id}/comments. Rate limit.

Comment → Mongo (comments): запись комментария {commentId, postId, authorId, text…}.

Comment → Event Bus: CommentAdded {postId, commentId…} (для уведомлений/аналитики).

Ответ клиенту.

Ленты обычно не пересобираются на комментарии, только на новые посты. Комменты подтягиваются при открытии поста.

Лайки и репосты (Like/Repost) — eventual consistency

Клиент → Gateway → Like/Repost: POST /posts/{id}/like или /repost. Rate limit.

Like/Repost → Mongo (aggregates): атомарный $inc счётчиков лайков/репостов по {postId}.

Like/Repost → Event Bus: LikeAdded / RepostCreated.

Ответ клиенту: UI может сразу показать «+1» локально.

Отложенная консистентность:

при чтении поста/ленты сервисы берут счётчик из aggregates;

если потребление событий задержалось, число может быть на пару секунд «старым» — это норма по требованию.

Чтение ленты новостей (Feed → timelines)

Клиент → Gateway → Feed: GET /timeline?cursor=…&limit=…. Rate limit.

Feed → Mongo (timelines): выборка готовых элементов для userId по индексу (userId, createdAt desc) с пагинацией (cursor).

Feed → Mongo (posts): батчем подтягивает сами посты по postId (и при необходимости — авторов из users).

Feed → Mongo (aggregates): подтягивает лайки/репосты батчем по postId.

Feed → Клиент: отдаёт элементы, курсор, агрегаты; ссылки на медиа — CDN-URL (быстро и дёшево).

Благодаря fan-out на запись чтение дешёвое и быстрое; это ключ к горизонтальному масштабированию.

Хранение мультимедиа (Media → Object Storage → CDN)

Сервис Media при загрузке пишет в Object Storage и возвращает objectKey.

CDN настроен поверх бакета/Origin; первые запросы могут попасть мимо кэша, затем CDN прогревается.

В постах хранится логическая ссылка на объект; на фронте мы строим CDN-URL.

Фоновые задачи Media могут делать трансформации (ресайз/видео-превью) и класть варианты рядом.

Rate limiting (Gateway + Redis)

Каждый входящий запрос в Gateway: ключ rate:{userId or ip}:{windowStart}.

Redis INCR + EXPIRE (скользящее окно или токен-бакет): если превышен лимит — 429 Too Many Requests.

Счётчики живут коротко (TTL 1–5 минут), чтобы не плодить мусор.

Кэш горячих данных (Redis)

Кэш профилей, постов «в тренде», последних N элементов ленты.

Invalidation по событиям (пост создан/удалён, профиль обновлён) или по TTL.

Не обязателен для корректности, лишь для ускорения.

Надёжность и масштабирование (как это работает под нагрузкой)

Stateless-сервисы (Auth/User/Social/Post/Comment/Like/Feed/Media) масштабируются горизонтально за счёт балансировки за Gateway.

Mongo: коллекции шардируются (обычно по userId или postId), реплика-сеты дают HA, чтения — с приоритетом на primary (или осторожно со secondaries там, где это допустимо).

Event Bus: несколько партиций по ключу (например, authorId), чтобы Fan-out обрабатывался параллельно, но упорядоченно по автору.

Feed: горизонтально масштабируется по followerRange/шардам; bulk-операции и idempotency (по postId + userId) для избежания дублей.

Observability: метрики RPS/latency/lag потребителей, доля ошибок, алерты по SLO.

Идемпотентность и отказоустойчивость (важно на практике)

Идемпотентные API для создания постов/лайков: клиент/сервер передают idempotencyKey → повторный запрос не создаст дубль.

Outbox-паттерн (по желанию) в Post/Comment/Like: запись события в таблицу-выход (Mongo коллекция) в одной транзакции с бизнес-записью → фоновый паблишер шлёт в Event Bus. Это минимизирует «записал пост, но событие не ушло».

Ретрии потребителей Event Bus с дедупликацией (по eventId).

Circuit breaker/timeout при обращениях к Mongo/Redis/Storage.
