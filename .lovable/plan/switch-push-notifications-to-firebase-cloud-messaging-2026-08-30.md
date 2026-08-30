# Switch push notifications to Firebase Cloud Messaging

Supabase keeps doing everything it does today: auth, database, storage, realtime, and the `notifications` table used for in-app history and live toasts. Only the *push delivery channel* changes from OneSignal to FCM.

## What exists today

- `src/integrations/onesignal/index.ts` — loads the OneSignal web SDK, asks permission, saves the subscription id into `onesignal_subscriptions`.
- `src/hooks/useNotifications.tsx` — registers the signed-in user for push, and subscribes to Supabase Realtime on `notifications` for in-app toasts.
- Two service-worker routes under `src/routes/api/public/OneSignalSDK*Worker.js.ts` plus copies in `public/`.
- Database: every insert into `notifications` fires trigger `notifications_dispatch_push` → `send_onesignal_push()`, which reads keys from `app_config`, collects the user's player ids, and POSTs to OneSignal via `pg_net`. Attempts are logged in `notification_dispatch_log`.
- Order-status changes already insert into `notifications` (via `notify_user` / `notify_role`), so nothing about order logic needs to change — only the delivery step.

## What will change

1. **Connect the Firebase Cloud Messaging connector** (with web push) so the service-account key stays server-side and the public web values arrive as environment variables. No credentials in source.
2. **New table `public.fcm_tokens`** — required, because FCM tokens are a different identifier from OneSignal player ids:
   `id uuid`, `user_id uuid → auth.users`, `token text unique`, `platform text default 'web'`, `user_agent text`, `last_seen_at timestamptz`, `created_at timestamptz`. RLS: users manage only their own rows; `service_role` full access. Existing tables are untouched (`onesignal_subscriptions` stays in place, unused, so nothing breaks).
3. **Client messaging module** `src/integrations/firebase/messaging.ts` — permission request from a user gesture, service-worker registration, `getToken` with the VAPID key, upsert into `fcm_tokens`, token-refresh handling, `onMessage` foreground handler that shows a toast, and token deletion on logout.
4. **Service worker** `public/firebase-messaging-sw.js` — receives background pushes (config passed via query string, since a worker cannot read Vite env).
5. **`src/hooks/useNotifications.tsx`** — swap the OneSignal register/logout calls for the FCM equivalents. Realtime in-app toasts stay exactly as they are.
6. **Server-side send path** — a server route `src/routes/api/public/fcm-dispatch.ts`, protected by a shared secret, sends the push through the Lovable connector gateway (`v1/projects/_/messages:send`). Firebase Admin credentials never reach the browser. The database trigger is repointed from `send_onesignal_push()` to a `send_fcm_push()` function that calls this route with `pg_net`, keeping the same `notification_dispatch_log` bookkeeping and the same "never block the insert" behaviour. Tokens that FCM reports as `UNREGISTERED`/invalid are deleted from `fcm_tokens`.
7. **Admin diagnostics** — `src/components/OneSignalDiagnostics.tsx` (shown at `/admin/notifications`) is replaced by an FCM diagnostics panel: permission state, worker registration, current token, stored token count, plus a "send test push" button.

## Notes

- Browsers block the permission prompt inside the Lovable preview iframe, so the UI will tell you to open the app in its own tab; production is unaffected.
- Nothing is removed from the `notifications` table, and no existing Supabase table or column is altered.
