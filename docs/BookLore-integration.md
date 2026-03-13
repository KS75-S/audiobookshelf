# BookLore ↔ Audiobookshelf Integration

This document outlines how to connect BookLore with Audiobookshelf (ABS) so that listening sessions and stats are shared between the two applications.

## Goals

- Allow BookLore to pull a user’s listening sessions and aggregated listening stats from ABS.
- Support JSON and CSV exports.
- Offer an optional webhook mechanism for real‑time updates (see "Push vs Pull").

## ABS API Endpoints

All URLs are relative to the ABS base URL, e.g. `http://localhost:13379/audiobookshelf/` in a local Docker setup.

### Authentication

ABS uses API keys. Create an API key in the server settings or use a user’s existing key. Supply it via an `Authorization: Bearer <key>` header.

### Sessions

| Method | Path                                | Description                        | Notes                |
| ------ | ----------------------------------- | ---------------------------------- | -------------------- |
| `GET`  | `/api/me/listening-sessions`        | Sessions for current user          | Paginated by default |
| `GET`  | `/api/users/:id/listening-sessions` | Sessions for any user (admin only) |

#### Query parameters (both endpoints)

- `itemsPerPage`, `page` – standard pagination. Defaults: 10/0.
- `all=1` – return _all_ sessions in one response (ignores pagination).
- `format=csv` – when combined with `all=1`, returns CSV instead of JSON.

**CSV output** columns: `id,libraryItemId,mediaItemId,episodeId,timeListening,currentTime,updatedAt,playMethod,deviceInfo[,user.id,user.username]`

#### Example URLs

```sh
# JSON dump of all sessions for current user
curl -H "Authorization: Bearer $KEY" \
    "http://localhost:13379/audiobookshelf/api/me/listening-sessions?all=1"

# CSV export for an admin fetching another user’s sessions
curl -H "Authorization: Bearer $ADMIN_KEY" \
    "http://localhost:13379/audiobookshelf/api/users/USER_ID/listening-sessions?all=1&format=csv"
```

### Stats

| Method | Path                             | Description                       |
| ------ | -------------------------------- | --------------------------------- |
| `GET`  | `/api/me/listening-stats`        | Aggregated stats for current user |
| `GET`  | `/api/users/:id/listening-stats` | Stats for another user (admin)    |

These endpoints accept an optional `year` query parameter to limit the stats to a calendar year.

## Push vs Pull

### Pull (recommended)

BookLore periodically polls the above endpoints (e.g. nightly) and ingests the returned JSON/CSV.

1. Store a user’s ABS API key in BookLore.
2. Fetch sessions and stats on a schedule.
3. Merge/append the data into BookLore’s own database.

This approach is simple and works over the public HTTP API.

### Push (optional)

If you prefer real‑time updates, ABS can call a webhook whenever a playback session is created.

- Add a webhook URL in ABS server settings.
- ABS will POST the new session object to that URL.
- BookLore handles the incoming payload and updates its records.

(Implementation of the webhook is not provided here but can be added by modifying `playbackSessionManager` to emit a request.)

## Notes & Tips

- The listen-history API returns raw ABS session objects. Fields such as `deviceInfo` are JSON blobs.
- The client‑side export buttons added to the ABS UI are purely convenience features and are not required for integration.
- Make sure the ABS container is rebuilt after code changes to expose the newest endpoints.
- If your ABS instance is behind a reverse proxy, include the `/audiobookshelf/` prefix in URLs.

## Troubleshooting

- **No sessions returned?** Verify the user ID and API key, and try querying with `?all=1`.
- **403 or 401 errors?** API key may lack required permissions; generate a new admin key if necessary.
- **Unusual CSV formatting?** Ensure the client handles quoted JSON fields.

**CORS errors?**

- Make sure the "Allowed CORS Origins" setting in ABS matches the exact origin of your client (e.g., `http://localhost:56800`).
- Do not include a trailing slash in the origin (use `http://localhost:56800`, not `http://localhost:56800/`).
- You must specify the port; wildcards like `http://localhost:*` are not supported.
- For development, you can use `*` to allow all origins, but this is not recommended for production.
- If you see a 401 Unauthorized on an OPTIONS request, ensure the ABS server allows unauthenticated OPTIONS requests for CORS preflight. Authentication should not be required for OPTIONS.
- After changing CORS settings, restart the ABS server/container if necessary.

---

With this README and the API hooks in place, BookLore can now keep your listening data in sync with Audiobookshelf effortlessly.

## How to Sync ABS Stats into BookLore

To sync a user’s Audiobookshelf (ABS) stats into BookLore, follow these steps:

1. **Obtain the user’s ABS API key**
   - The user can generate or retrieve their API key from the ABS server settings.

2. **Fetch stats using the ABS API**
   - Use the following endpoint to get aggregated listening stats for the current user:
     - `GET /api/me/listening-stats`
   - Optionally, add `?year=YYYY` to limit stats to a specific year.
   - For admins, fetch stats for any user:
     - `GET /api/users/:id/listening-stats`

   **Example request:**

   ```sh
   curl -H "Authorization: Bearer <API_KEY>" \
         "http://<abs-url>/api/me/listening-stats"
   ```

3. **Store the API key in BookLore**
   - BookLore should securely store the user’s ABS API key for scheduled syncs.

4. **Schedule periodic syncs**
   - BookLore should poll the stats endpoint on a regular schedule (e.g., nightly) to fetch the latest stats.

5. **Ingest and merge stats**
   - Parse the JSON response and merge or append the stats into BookLore’s database.

6. **(Optional) Real-time updates**
   - For real-time syncing, implement a webhook in BookLore and configure ABS to POST new session objects to your webhook URL (requires ABS server modification).

Refer to the "Stats" and "Pull (recommended)" sections above for more details. If you need code samples or further implementation help, see the API endpoint documentation or reach out to the maintainers.
