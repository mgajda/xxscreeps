**Status**: DRAFT — for filing upstream (`github.com/laverdet/xxscreeps` issues).
**Drafted**: 2026-10-09.
**Source**: `tmp/xxscreeps-bot-remove-fix.md` (hand-off from the `screeps` class-server session).

# `manage bot remove` deletes the account, not the bot's code

## Summary

`xxscreeps manage bot remove <name|id>` runs the same handler as
`user remove`, so it deletes the whole **account**: provider links, the
`users` set entry, the code, and every mod's per-user state. It should
delete only the bot's code.

## Where

`packages/xxscreeps/scripts/manage.ts:492`:

```ts
case 'bot remove': if (rest[0] === undefined) usage(); await userRemove(rest[0]); break;
```

`userRemove` is the account-deleting verb. Its siblings do the right
thing:

```ts
case 'bot add':    await botSave(rest[0], rest[1], rest[2], true);  break;
case 'bot update': await botSave(rest[0], rest[1], rest[2], false); break;
case 'bot remove': if (rest[0] === undefined) usage();
                   await userRemove(rest[0]);                       break;   // ← wrong
```

The compiled copy that ships in the image is
`node_modules/xxscreeps/dist/scripts/manage.js:522`.

## Impact

`userRemove` calls `User.remove` (`packages/xxscreeps/engine/db/user/index.ts`),
which deletes, for the user id:

- `user/<id>/provider` (the provider links)
- `user/<id>/branches` and every branch's `.../bins/<branch>` /
  `.../code/<branch>` blobs
- the `users` set entry and `usersByProvider/<provider>/<id>` entries
- mod-owned per-user state via `remove` hooks (mail, leaderboard,
  decorations, …)

After that the account is gone: it cannot sign in, and any password row
in an inventory matches nothing. **Its rooms are also left behind** — the
spawn and the claimed controller survive, owned by a user id that no
longer exists ("owned by a ghost"). In the reporting project a block room
became unusable until it was regenerated, because nobody existed to
release the claim.

## Reproduction

About 40 s per CLI call; the `manage` verb boots the whole engine.

```sh
xxscreeps manage user create zz_probe
xxscreeps manage user list | grep zz_probe        # present
xxscreeps manage bot remove zz_probe
xxscreeps manage user list | grep zz_probe        # BUG: gone
```

A regression test that exercises exactly this is
`tests/cli-does-not-delete-accounts.sh` in the reporting project: it
creates one throwaway account, asks the CLI to remove a *bot*, and
requires the account to survive.

## Suggested fix

A `bot remove` should delete only the code, i.e. the code subset of
`User.remove`:

1. Export a `removeCode(db, userId)` from
   `packages/xxscreeps/engine/db/user/code.ts` that deletes the branch
   manifest and each branch's buffers/strings, reusing that module's
   existing `branchManifestKey` / `buffersKey` / `stringsKey` helpers.
2. Have `User.remove` call `removeCode` for its code portion (so the two
   key lists cannot drift).
3. Route `manage bot remove` at a code-only handler; keep `user remove`
   as the documented destructive verb.

## Separate observation (maybe its own issue)

`user remove` leaves the account's rooms (spawn / controller) orphaned,
so recovery needs the room regenerated. That is distinct from the
`bot remove` defect and could be filed separately if preferred.
