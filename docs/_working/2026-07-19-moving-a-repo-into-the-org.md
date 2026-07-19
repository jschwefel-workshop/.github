<!-- DRAFT — working copy. Seal by `git mv` into docs/ (drop the date prefix) once approved. -->

# Moving a repo into `jschwefel-workshop`

A repeatable runbook for transferring an existing repository into the `jschwefel-workshop`
organization and settling it in — on GitHub and on disk. Written to work for any repo (kiln,
esp-idf-ds3231, and whatever comes next), not tied to one project.

## What you need first

- **Owner** (admin) on `jschwefel-workshop`. Both `jschwefel` and `jschwefel-CBB` are
  owners, so either account qualifies as the *destination* owner.
- **Admin on the source repo.** The transfer is driven by whoever owns the repo **today** —
  the source account, not the destination:
  - a repo under `jschwefel-CBB/…` → run the transfer **as `jschwefel-CBB`**
  - a repo under `jschwefel/…` (personal account) → run it **as `jschwefel`**
  You cannot transfer a repo you don't have admin on.
- The target name must be **free** in the org — no repo of that name already there:
  ```sh
  gh api /repos/jschwefel-workshop/<repo> -q .full_name   # want a 404 (Not Found)
  ```

## 1. Transfer on GitHub

`gh` has no `repo transfer` command — it's REST only:

```sh
gh api -X POST /repos/<current-owner>/<repo>/transfer -f new_owner=jschwefel-workshop
```

Returns **202 Accepted** and completes **asynchronously**. Poll until it lands:

```sh
until [ "$(gh api /repos/jschwefel-workshop/<repo> -q .full_name 2>/dev/null)" = "jschwefel-workshop/<repo>" ]; do sleep 2; done
echo "landed"
```

(UI path: repo → **Settings** → **Danger Zone** → **Transfer ownership**.)

## 2. Verify the move

```sh
gh api /repos/jschwefel-workshop/<repo> -q '.full_name + "  (owner=" + .owner.login + ")"'
```

- Issues, PRs, **releases, tags**, stars, watchers, and the wiki move **with** the repo.
- The old path (`<current-owner>/<repo>`) now **redirects** — git, web, and API all follow it.
  Treat the redirect as a courtesy, not a contract.
- **Never create a new repo of the old name under the old account** — that permanently kills
  the redirect.

## 3. Update in-repo references

Redirects cover old links, but repoint anything that hardcodes the old path so the repo is
self-consistent:

```sh
grep -rn "<current-owner>/<repo>" . --exclude-dir=.git
# → replace with jschwefel-workshop/<repo> in README, docs, CI badges, release/install URLs
```

Branch → PR → CI → merge, like any change.

## 4. Re-home the local clone

Match the on-disk pattern — each org is a directory; its repos live inside it:

```sh
mv ~/repositories/<repo> ~/repositories/jschwefel-workshop/<repo>
cd ~/repositories/jschwefel-workshop/<repo>
git remote set-url origin git@github.com:jschwefel-workshop/<repo>.git   # only if origin is the GitHub remote
git fetch origin   # confirm the new remote answers
```

If the clone tracks some **other** remote (a self-hosted mirror, a different `origin`), leave
that alone — only a GitHub `origin` pointing at the old owner needs changing.

## 5. Post-move checks

- **CI / Actions** run under the new owner automatically. But **Actions secrets do not always
  travel** — re-add any secret the workflow depends on. (A `swift build && swift test` CI with
  no secrets needs nothing.)
- **Branch protection, webhooks, deploy keys** *should* move, but verify anything load-bearing
  rather than trusting it.
- **Homebrew tap** — only if the repo is a `homebrew-*` tap: the tap path becomes
  `jschwefel-workshop/<name>`; re-tap locally and update the install docs. (N/A for kiln and
  esp-idf-ds3231.)

## 6. Keep the org home page current

The org README lives at `jschwefel-workshop/.github` → `profile/README.md`. It already lists
the incoming repos with live links, so a link resolves the moment its repo lands. **Any repo
you add later that isn't already listed → add it to that README in the same change.** A stale
org front page is the first thing every visitor sees, and nothing warns you it rotted.

---

## For the current batch

- **`kiln`** — source is `jschwefel-CBB/kiln`, so **`jschwefel-CBB` runs the transfer**. It's
  source-available (BSL 1.1); that's fine — the org deliberately holds mixed licensing. Its
  local clone tracks a self-hosted remote (`deb005`), not a GitHub `origin`, so step 4 is just
  the directory move unless you add a GitHub remote.
- **`esp-idf-ds3231`** — source is on the **personal `jschwefel` account**, so **that account
  initiates the transfer** (`jschwefel-CBB` can't transfer a repo it doesn't own). Everything
  else is identical.

Both are already linked from the org home page, so each link goes live the instant its
transfer lands.
