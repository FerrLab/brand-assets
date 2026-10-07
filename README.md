# FerrLab brand assets

Public host for **approved** FerrLab marketing creatives.

It exists for one reason: a third-party server — Postiz, an ad platform, a press
contact's CMS — can only ingest an image if it can fetch the bytes itself, from a
URL, with no credentials. A file that lives only as a Paperclip attachment has no
such URL, so until now every creative FerrLab composited itself needed the founder
to upload it by hand. A file committed here has one:

```
https://raw.githubusercontent.com/FerrLab/brand-assets/main/<path>
```

## Rules — read before you commit

1. **Everything here is public, for ever.** Git keeps history; a deleted file is
   still in the repository. Treat every commit as a publication.
2. **Approved creatives only.** The founder, or the approver named on the issue,
   must have approved the exact file for publication before it is pushed. No
   drafts, no work-in-progress, no rejected variants.
3. **Never commit customer or tenant data.** No tenant names, no pilot names, no
   screenshots of a live tenant's data, no internal metrics, no credentials.
4. **Source art does not belong here.** Only the final, flattened, publishable
   file. Keep layered sources and intermediates as Paperclip attachments.
5. **Never force-push and never rewrite history.** A published URL must keep
   resolving; a social post that already cites one breaks if the blob moves.

## Layout

```
creatives/<year>/<month>/<descriptive-name>.<ext>
```

Name the file for what it depicts and where it goes, not for the issue that
caused it — the issue number belongs in the commit message, where it stays
useful without pinning the asset to one ticket. Include the pixel dimensions when
the aspect ratio is what makes the asset what it is: a 1080x1920 story frame is
not interchangeable with a 1080x1080 feed post.

## Runbook: get a self-composited creative into Postiz

Run this from any agent shell. The bytes move through `curl` and `git`, never
through a tool argument, so the ~2.5M-character tool-argument limit does not
apply — that limit is what blocked every earlier attempt, and it is not a
byte-transport limit.

```bash
. /root/.paperclip/tools/env.sh          # CA bundle + toolchain; without it, curl exits 77
B="${PAPERCLIP_API_URL%/}"; B="${B%/api}"
D="$PAPERCLIP_RUN_SCRATCH_DIR"

# 1. attachment bytes to disk (no encoding step)
curl -sfS -H "Authorization: Bearer $PAPERCLIP_API_KEY" \
  "$B/api/attachments/<attachment-id>/content" -o "$D/asset.png"
sha256sum "$D/asset.png"                 # compare with the hash recorded on the issue

# 2. publish
git clone -q "https://x-access-token:${PAPERCLIP_GIT_TOKEN}@github.com/FerrLab/brand-assets.git" "$D/ba"
mkdir -p "$D/ba/creatives/$(date +%Y/%m)"
cp "$D/asset.png" "$D/ba/creatives/$(date +%Y/%m)/<descriptive-name>.png"
git -C "$D/ba" add -A
git -C "$D/ba" commit -q -m "Add <what it is> (FER-nnn)"
git -C "$D/ba" push -q origin main

# 3. confirm the public URL serves the same bytes, unauthenticated
URL="https://raw.githubusercontent.com/FerrLab/brand-assets/main/creatives/$(date +%Y/%m)/<descriptive-name>.png"
curl -sfS "$URL" | sha256sum             # must match step 1
```

Then hand `$URL` to Postiz's `uploadfromurltool`, which makes Postiz's own server
fetch it into the media library, and use the returned media id as the attachment
on `integrationscheduleposttool`.

Verify the hash in step 3 before you schedule anything. `raw.githubusercontent.com`
serves from a cache and has no SLA; it is fine for a one-time server-side fetch,
but a 404 at schedule time is better found by you than by the platform.
