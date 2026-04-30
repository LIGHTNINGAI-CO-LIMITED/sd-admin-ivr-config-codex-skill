---
name: sd-admin-ivr-config
description: Use when configuring, creating, updating, validating, or troubleshooting IVR/script-configuration workflows in the Shandian Intelligent admin backend at ai.sd6g.com:1904, especially tasks involving 话术配置, 智能Agent/智能节点, /api/web IVR APIs, sceneList/sceneListFrontend, prompt Markdown import, or direct Bearer token API calls without logging in.
---

# Shandian Admin IVR Config

Use this skill to operate the Shandian Intelligent admin backend by API, not by logging into the page. The default path is: the user provides a valid `Bearer` token, validate it with `/account/findInfo`, then call `/api/web` endpoints directly.

## Non-Negotiables

- Do not use the login page or captcha flow unless there is no valid token and the user explicitly asks for login troubleshooting.
- Do not print, store in docs, or repeat real tokens/passwords in final answers.
- Use request header `token: Bearer <TOKEN>`, not `Authorization`.
- Verify token first with `GET /account/findInfo`; continue only when `code=0`.
- For write operations, create or use a test/new IVR unless the user explicitly asks to modify an existing production IVR.
- For `updateSceneList`, snapshot existing config first and verify by reading it back.

## Quick Workflow

1. Extract token from the user's curl or message.
   - Prefer `-H 'token: Bearer ...'`.
   - If only cookie is present, URL decode `token=Bearer%20...`.
2. Validate:
   - `GET https://ai.sd6g.com:1904/api/web/account/findInfo`
   - Header: `token: Bearer <TOKEN>`
3. Read base resources:
   - `GET /industry/findList`
   - `GET /ivr/findAllTtsVoiceBaseInfo`
   - `GET /ivr/findModelList`
   - `POST /ivr/findPage` with `{"query":{"searchName":""},"page":{"current":1,"size":10}}`
4. Create IVR with `/ivr/insert`, or read the target IVR if updating.
5. For smart Agent nodes, clone a known-good scene graph shape from an existing IVR, then replace only business fields.
6. Import prompt Markdown as UTF-8. If the full prompt causes `话术场景信息异常`, compact whitespace and retry under the backend's practical prompt length.
7. Verify with `/ivr/findSceneList/{ivrId}` and, when useful, open `/script-graph?ivrId=<ivrId>`.
8. Delete temporary token files or auth dumps.

## Key API Rules

- Base URL: `https://ai.sd6g.com:1904/api/web`
- IVR pagination payload must be:

```json
{"query":{"searchName":""},"page":{"current":1,"size":10}}
```

- New IVR minimal payload:

```json
{"voiceType":1,"ttsVoiceId":1,"speechRate":1,"name":"<name>","industryId":42}
```

- Smart Agent node type is `type: 4`.
- Agent model config lives at `llmNodeModelConfig`.
- Keep backend and frontend copies in sync:
  - `sceneList[0].nodeList[0]`
  - `sceneListFrontend[0].nodeList[0]`
  - `sceneListFrontend[0].graph.cells[0].data.customData`

## References

Read `references/workflow.md` when you need the detailed end-to-end procedure, payload templates, prompt-length workaround, validation checklist, or troubleshooting notes.
