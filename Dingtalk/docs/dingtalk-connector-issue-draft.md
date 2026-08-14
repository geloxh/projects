## Title
Probe fails with 403 on getAccessToken despite valid, verified credentials

## Environment
- OpenClaw version: 2026.7.1-2 (0790d9f)
- dingtalk-connector version: 0.8.24
- OS: Windows 11 via WSL2 (Ubuntu), systemd user service
- Node.js: v26.7.0

## Symptom
`openclaw channels status --probe` consistently returns:
```
- DingTalk __default__: enabled, configured, running, connected, probe failed, error:Request failed with status code 403
```

The channel shows `connected` (Stream WebSocket succeeds), but the probe (which calls `getAccessToken(config)` per plugin.ts) fails with a bare 403 and no response body.

## What I've already verified
1. **Credentials are valid** — confirmed via direct curl against DingTalk's own OAuth endpoint:
   ```
   curl -X POST 'https://api.dingtalk.com/v1.0/oauth2/accessToken' -H 'Content-Type: application/json' -d '{"appKey": "****************", "appSecret": "<redacted>"}'
   ```
   Response: `{"expireIn":7200,"accessToken":"..."}` — success.

2. **Config values match exactly, byte-for-byte** — verified via `xxd` on both the raw `openclaw.json` clientSecret and the value used in the successful curl call above. No whitespace/encoding mismatch.

3. **App is published** — Version Management shows 3 versions, current one is "On the line" (线上版本/live).

4. **Robot capability is set to Stream mode**, not HTTP/Webhook.

5. **No IP whitelist configured** on the app (empty list — should mean unrestricted).

6. **Relevant permissions enabled** (all show "Opened" status):
   - In-company robot messaging permission (企业内机器人发送消息权限)
   - Interactive card instance write permissions (Card.Instance.Write)
   - AI card streaming update permissions (Card.Streaming.Write)
   - Intelligent interactive card writing permissions

7. **No IP/env override present** — checked systemd service file (`systemctl --user cat openclaw-gateway.service`) for stray env vars; none found related to DingTalk/DWS.

8. Config passes validation (no "additional properties" errors) using only the documented fields: `enabled`, `clientId`, `clientSecret`, `dmPolicy`, `groupPolicy`, `requireMention`, `separateSessionByConversation`, `sharedMemoryAcrossConversations`, `groupSessionScope`.

## One anomaly noticed
Gateway startup log shows:
```
starting dingtalk-connector[__default__] (mode: stream, DINGTALK_AGENT=DING_DWS_CLAW, DWS_CLIENT_ID=dingyxxx...)
```
Not sure if `DINGTALK_AGENT=DING_DWS_CLAW` indicates the connector is internally routing through DEAP Agent / DWS logic rather than plain bot mode (方案一), even though my config only specifies plain bot fields. Could this be relevant to the 403?

## Question
What specific DingTalk API call is failing with 403 here (probe just reports the bare HTTP status, no body)? Is there a permission or app-type requirement not covered in the current TROUBLESHOOTING.md that would explain a 403 specifically on `getAccessToken`/probe, given a token was independently confirmed to be obtainable via direct API call with the exact same credentials?

Happy to provide `openclaw logs --follow` output during a live probe run, or any other diagnostic info needed.
