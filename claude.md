# AI Studio: guide for Claude

> للشريك: قول لـ Claude Code بتاعك: «اقرا https://rioworkv5-bit.github.io/studio/claude.md واشتغل على الاستوديو».
> ومفتاحك حطه في متغيّر بيئة اسمه `STUDIO_TOKEN`. ما تكتبهوش في الشات ولا في أي ملف بيترفع على GitHub.

You are helping a member of the AI Studio team. AI Studio is an Egyptian drama/ads production app that runs on the owner's
computer: script → characters/locations → shot pictures → voice → video → assembled scene. Its interface is Egyptian Arabic.
Talk to the user in Egyptian Arabic.

## Connect
1. The studio's address changes every time its computer restarts. Read the current one each session:
   `curl -s https://rioworkv5-bit.github.io/studio/studio.json` → `{"url": "https://….trycloudflare.com", ...}`.
   If a request fails to connect, read it again: the studio may have restarted. If it still fails, the studio is off; tell the user.
2. Every request carries the user's key from the environment: `-H "Authorization: Bearer $STUDIO_TOKEN"`.
   Never print the key, never write it into a file, never ask the user to paste it into the chat.
   If `STUDIO_TOKEN` is not set, ask the user to set it (PowerShell: `setx STUDIO_TOKEN "<key>"`, then open a new terminal).
3. Check: `GET /api/me` → `{"name": "<email>", "role": "partner"}`.

```bash
S=$(curl -s https://rioworkv5-bit.github.io/studio/studio.json | node -e "process.stdin.on('data',d=>console.log(JSON.parse(d).url))")
A="Authorization: Bearer $STUDIO_TOKEN"
curl -s -H "$A" "$S/api/me"
```

## What a partner can do
- See and edit the **public** projects (private ones are hidden from you).
- **Pictures are made by the user on his own Google Flow, then uploaded.** The studio does not generate pictures for a partner:
  generating a frame or a character/location image is refused. Never try to get around this.
- Generate **voice** and **video**, put a scene together.

## API (JSON; base = `$S`)
- `GET /api/projects` → list. `GET /api/p/<pid>` → project: `characters`, `locations`, `scenes`, `shots` (in order).
  A shot: `id`, `sceneId`, `desc`, `dialogue`, `speakerId`, `frame` (current picture file), `frames`, `frameOk`
  (= `frame` when the picture is approved), `video`, `videoOk`, `lineOk`.
- Edit: `PATCH /api/p/<pid>/shots/<id>` (also `characters`, `locations`, `scenes`) with only the fields to change.
- Upload a picture (png/jpg/webp): `PUT /api/p/<pid>/upload?ext=png` with the raw file as body → `{"file": "u_….png"}`. Then
  - shot: `PATCH …/shots/<id>` `{"frames": [...shot.frames, file], "frame": file}`
  - character/location: `PATCH …/characters/<id>` `{"images": [file, ...old images]}` (the first image is the one used).
- Voice of a shot's line: `POST …/shots/<id>/gen-line` (needs `speakerId`); progress in `shot.lineJob.status`.
- Video: `POST …/shots/<id>/generate` `{"kind": "video"}`. Needs an approved picture (`frameOk === frame`).
- Assemble a scene (shots back to back with their voices): `POST …/scenes/<sceneId>/assemble` → `scene.animatic`.
- Files: `GET $S/files/<pid>/<file>` (with the same header).
- Arabic text in a body: write the JSON to a UTF-8 file and send it with `--data-binary @file`
  (inline Arabic on some Windows shells turns into `?`).

## Rules
- Approving («✓ تمام»: `frameOk`, `videoOk`, `lineOk`, `ok`) is the human's decision. Don't set them unless the user tells you
  to approve that exact item after looking at it. Never approve many at once.
- Don't edit, delete or regenerate in a project the user didn't ask you to work on.
- Generation runs on one shared computer: send one request, wait for it, don't flood the queue, don't cancel others' jobs.
- You can't judge acting, tone or pronunciation by ear: ask the user to listen.
- Report what you changed (project, scene, shot number) so the team can find it in the studio.
