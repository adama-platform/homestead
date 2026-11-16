# Hound cases
Integration tests for hound against HEVY and Strava, as Postman collections
------

> Status: examples written with the design, before hound exists. They run once `hound_main` is built, and they open in Postman today.

## What these are

Each file is a Postman Collection v2.1 with a `hound` extension that Postman ignores. Run with `hound run`, every item marked with a `binding` goes through the same pipeline barn uses for a box (`hound_policy`, then `hound_auth`, then `hound_client`), so a pass here says the production code path works against the real provider. Opened in Postman, the same requests run with Postman's own auth, which is a quick way to tell a provider problem from a hound problem.

| file | validates |
| --- | --- |
| `hevy.postman.json` | An `api-key` credential attached in a header; paging; captures across requests; a write (creating a routine) behind `--allow-writes`; six policy refusals. |
| `hevy.env.example.json` | The variables HEVY needs: your API key. |
| `strava.postman.json` | An `oauth2-user` credential with a rotating refresh token; capturing a secret; rate-limit headers; five reads; seven policy refusals. |
| `strava.env.example.json` | Your Strava app's client id and secret, and a refresh token for one athlete. |

## Running them

Copy an example environment, fill it in, and keep the copy out of git (`*.env.json` is meant to be ignored):

```
cp hevy.env.example.json hevy.env.json            # set hevy_api_key
hound run hevy.postman.json --env hevy.env.json
hound run hevy.postman.json --env hevy.env.json --folder policy          # offline: nothing is sent
hound run hevy.postman.json --env hevy.env.json --allow-writes           # also creates a routine

cp strava.env.example.json strava.env.json        # set strava_app.*, strava_refresh_token
hound run strava.postman.json --env strava.env.json
hound run strava.postman.json --env strava.env.json --record recordings/strava
hound run strava.postman.json --replay recordings/strava                 # CI: no network, no secrets
```

Secrets can come from the process environment instead of a file: `HOUND_HEVY_API_KEY=... hound run hevy.postman.json`.

- **HEVY** needs a Pro account; the API key is in HEVY's settings under the developer section.
- **Strava** needs an API application (client id and secret) and a refresh token for an athlete who authorized it with at least `read` and `activity:read_all` (`profile:read_all` for zones). The `auth` folder refreshes the token first and captures the new pair, since Strava rotates refresh tokens: after a run, the environment's refresh token may be stale, and `hound run` writes captured secrets back to the environment file only with `--save-env`.

## How credentials are found

A binding names a secret like `tenant/hevy_api_key`. In barn, cellar answers it. In the CLI, the environment does:

| secret | looked up as |
| --- | --- |
| `tenant/hevy_api_key` | the variable `hevy_api_key` |
| `tenant/strava_app` | the variables starting `strava_app.` (`client_id`, `client_secret`) |
| oauth2-user tokens | `strava_access_token`, `strava_refresh_token`, `strava_expires_at` (the name of the binding, then `_access_token`, and so on), standing in for pond's `_credentials` bucket |

## What each kind of item proves

| kind | passes when |
| --- | --- |
| A read through a binding | The policy allowed it, hound attached the credential (the item sets none), the provider answered, and the expectations hold. |
| A write | As a read, and only with `--allow-writes`. Without it, the item is skipped and counted. |
| A policy item | Hound refused it locally with exactly the named code and sent nothing. Policy items need no credentials and no network, so `--folder policy` is a fast offline check of `hound_policy`. |
| The Strava `auth` folder | Strava's token endpoint answered the refresh the way `hound_auth` expects it to. |

## Verify against the provider's docs

These were written from the providers' documentation as best understood, without running them. Before trusting a failure, check:

- **HEVY** (https://api.hevyapp.com/docs): the paging parameter names (`page`, `pageSize`), the response keys (`workout_count`, `page_count`, `workouts`, `events`, `routines`, `routine_folders`, `exercise_templates`), and the body of `POST /v1/routines` (`routine.exercises[].sets[]`, `exercise_template_id`, `rest_seconds`, `superset_id`). Write rate limits and Pro requirements may change.
- **Strava** (https://developers.strava.com/docs/reference/): `sport_type` versus the older `type` on activities, the keys of `/athletes/{id}/stats` (`all_run_totals`, `recent_ride_totals`), `/athlete/zones` (`heart_rate`), and the rate-limit headers. Strava has reported `X-RateLimit-Limit` and `X-RateLimit-Usage` as two comma-separated windows (15 minutes, daily) and has added read-specific limits (`X-ReadRateLimit-*`); the binding's `100/15m` is a guess at the read limit for a default app.

When a field turns out wrong, fix the expectation here; the cases are documentation of what hound and a box can rely on.

## Later

`hound import` will turn an OpenAPI spec into a collection like these, with `schema` expectations. Strava publishes a Swagger 2 spec, which would need converting to OpenAPI 3 first; HEVY's docs are Swagger UI, and whether a spec file is downloadable needs checking. JetBrains and VS Code `.http` files, and Hurl files, are candidates for later imports.
