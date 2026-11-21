# Click Counter

Independent service that counts clicks for [shrink](../shrink) links. It uses the same Postgres database and the same `links` / `link_clicks` tables, and counts clicks two ways:

1. **Kafka consumer**: reads `link.clicks` events published by shrink's redirect and counts each one.
2. **HTTP API**: `POST /api/v1/links/:slug/click` counts one click for a link.

Both go through one function (`recordClick` in `src/db.ts`). In a single transaction it inserts a `link_clicks` row and does `links.clicks = clicks + 1`. `link_clicks.event_id` is unique, so a redelivered Kafka message or a retried API call with the same `eventId` is counted once.

## Quick start

```bash
npm install
cp .env.example .env   # fill in DATABASE_URL, API_KEYS, KAFKA_BROKERS
npm run dev            # http://localhost:4000
```

| Command | What it does |
|---|---|
| `npm run dev` | Start with auto-reload |
| `npm run build` / `npm start` | Compile and run |
| `npm run typecheck` | Type-check |

Set `KAFKA_ENABLED=false` to run the HTTP API without Kafka. The server refuses to start without `DATABASE_URL` and `API_KEYS`.

## API

Auth on `/api/*`: `x-api-key: <key>` or `Authorization: Bearer <key>`.

### `GET /health`
No auth. `{ "status": "ok" }`, or `503` if the database is unreachable.

### `POST /api/v1/links/:slug/click`

```bash
curl -X POST http://localhost:4000/api/v1/links/abc1234/click \
  -H "x-api-key: $API_KEY" -H "Content-Type: application/json" \
  -d '{"eventId":"7b0f0c2e-5a7e-4a52-9a53-2d9f1f3c1a11"}'   # body optional
```

| Status | Body |
|---|---|
| `200` | `{ "counted": true, "slug": "abc1234", "clicks": 42 }` |
| `200` | `{ "counted": false, "reason": "duplicate_event" }` (same `eventId` seen before) |
| `400` | Invalid body (only an optional UUID `eventId` is accepted) |
| `401` | Missing or invalid API key |
| `404` | No active link with that slug |

Without an `eventId`, every call counts. Send one if the caller may retry.

## Notes

- Kafka messages are `{ eventId, linkId, at }` (see `shrink/lib/kafka.ts`), and the consumer group is `sh-counter`. Messages that fail to parse are skipped. DB errors make kafkajs retry the message.
- Shrink's redirect still falls back to counting directly in the DB when Kafka is down. That path has no `eventId` and does not conflict with this service.
- If you run more than one instance, they share the consumer group and split the partitions.
