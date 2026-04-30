# Shandian Admin IVR Workflow Reference

## Auth

Default: do not login.

Use:

```http
token: Bearer <TOKEN>
```

Validate before doing anything:

```http
GET https://ai.sd6g.com:1904/api/web/account/findInfo
```

Expected success:

```json
{"code":"0","message":"success","data":{...}}
```

If `code=7`, the token is invalid or expired. Ask for a fresh token. Do not jump to captcha login unless the user asks.

## Common PowerShell Request Pattern

```powershell
$base = 'https://ai.sd6g.com:1904/api/web'
$headers = @{ token = 'Bearer <TOKEN>'; 'X-Requested-With' = 'XMLHttpRequest' }
$info = Invoke-RestMethod -Method Get -Uri "$base/account/findInfo" -Headers $headers
```

For JSON POST:

```powershell
$body = @{ query = @{ searchName = '' }; page = @{ current = 1; size = 10 } } |
  ConvertTo-Json -Depth 10 -Compress
$r = Invoke-RestMethod -Method Post -Uri "$base/ivr/findPage" `
  -Headers $headers -ContentType 'application/json;charset=UTF-8' -Body $body
```

## Base Reads

Call these before writing:

```http
GET  /industry/findList
GET  /ivr/findAllTtsVoiceBaseInfo
GET  /ivr/findModelList
POST /ivr/findPage
POST /ivrGroup/findPageIvrGroup
```

`/ivr/findPage` uses:

```json
{"query":{"searchName":""},"page":{"current":1,"size":10}}
```

Do not use plain `{"current":1,"size":10}`; it returns `illegal argument`.

## Create IVR

Endpoint:

```http
POST /ivr/insert
```

Minimal payload:

```json
{
  "voiceType": 1,
  "ttsVoiceId": 1,
  "speechRate": 1,
  "name": "<话术名称>",
  "industryId": 42
}
```

Notes:

- `industryId=42` is `教育/K12-其他`; change if the task requires another industry.
- `ttsVoiceId=1` is safe as a default when no voice is specified.
- The response may directly return the new `ivrId`.
- Confirm with `/ivr/findPage` by name.

## Scene Graph Strategy

New IVRs may have `sceneList=null` and `sceneListFrontend=null`.

Do not hand-roll the entire graph if a template exists. Safer pattern:

1. Find an existing IVR with a known-good one-scene smart Agent graph.
2. `GET /ivr/findSceneList/{templateIvrId}`.
3. Copy `data.sceneList` and `data.sceneListFrontend`.
4. Write those strings unchanged to the new IVR with `/ivr/updateSceneList`.
5. If unchanged write succeeds, update business fields in small batches.

Endpoint:

```http
POST /ivr/updateSceneList
```

Payload:

```json
{
  "ivrId": 3449,
  "sceneList": "<JSON string>",
  "sceneListFrontend": "<JSON string>"
}
```

## Smart Agent Node Fields

Smart Agent node:

```json
{
  "type": 4,
  "isStartNode": true,
  "name": "<节点名>",
  "text": "<开场白>",
  "allowInterruptEnabled": true,
  "allowInterruptSecond": 2,
  "llmNodeMaxSpeakRound": 60,
  "llmNodeModelTimeoutMilliSecond": 1000,
  "llmNodeModelConfig": {
    "id": 55,
    "prompt": "<prompt>",
    "enableThinking": 0,
    "enable_thinking": 0
  }
}
```

Keep these synchronized:

- Backend node: `sceneList[0].nodeList[0]`
- Frontend node: `sceneListFrontend[0].nodeList[0]`
- Graph custom data: `sceneListFrontend[0].graph.cells[0].data.customData`
- Graph display fields:
  - `data.label`
  - `data.title`
  - `data.description`

## Prompt Import

Read Markdown prompts with UTF-8.

If full prompt write fails with `话术场景信息异常`, test whether a short prompt saves. If short prompt saves, compact the prompt:

- Remove code fences.
- Remove blank lines.
- Trim trailing spaces.
- Preserve all business wording, intent labels, and rules.

Observed practical limit: a 10418-character prompt failed; a 9686-character compact version saved.

## Validation Checklist

After writing:

```http
GET /ivr/findSceneList/{ivrId}
```

Check:

- `code=0`
- `sceneList` and `sceneListFrontend` are non-null.
- Scene name is expected.
- Smart node has `type=4`.
- Node is start node if intended.
- `llmNodeModelConfig.id` is expected.
- Prompt length/hash matches the local compact prompt.
- Prompt matches in backend node, frontend node, and graph custom data.

Optional browser check:

```text
https://ai.sd6g.com:1904/script-graph?ivrId=<ivrId>
```

## Troubleshooting

- `code=7`: token invalid/expired. Ask for a fresh token.
- `illegal argument` on `/ivr/findPage`: use nested `query` and `page`.
- `话术场景信息异常`: graph structure invalid, frontend/backend copies diverged, or prompt too long.
- Mojibake/question marks in Chinese fields: do not embed Chinese literals in a non-UTF-8 shell script; pass them through UTF-8 files or PowerShell `ConvertTo-Json`.
- Page redirects to `/login`: browser cookie is missing/expired, but API may still work if Header token is valid.

## Security

- Never include real token values in docs or final responses.
- Delete temporary auth files after work.
- Avoid saving browser cookie dumps in the workspace.
