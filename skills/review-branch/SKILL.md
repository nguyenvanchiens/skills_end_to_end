---
name: review-branch
description: Deprecated pointer. Use when someone invokes review-branch — the whole-branch review workflow now lives in the gitlab-review skill.
---

# review-branch — moved into `gitlab-review`

> ⚠️ **Deprecated as a standalone skill.** The whole-branch review workflow that used to live here now lives in **one canonical place**: the `gitlab-review` skill, under the **`review the whole branch`** trigger. Same move the `commit` skill made into `gitlab-flow`.

Keeping two copies caused real drift: this file went without a severity model entirely while `gitlab-flow` had carried one since commit `733261d`, and a single fix later had to be applied twice, in two languages. One canonical home is the fix.

## Use this instead

```bash
npx skills add nguyenvanchiens/skills_end_to_end -s gitlab-flow -y -a claude-code --copy
npx skills add nguyenvanchiens/skills_end_to_end -s gitlab-review -y -a claude-code --copy
```

`gitlab-review` **depends on** `gitlab-flow` — install both. It reads the severity model, review lenses, base-branch rules, and output-language rules from `gitlab-flow` rather than copying them.

Then type `review the whole branch`. The workflow is unchanged: four specialized agents in parallel → dedup + verify → auto-fix the `Blocker`/`Major` findings that survive verification.

## Not using GitLab / `glab`?

`review the whole branch` only needs `git`. `glab` appears in one optional branch of base-branch detection, which you can skip. The other three `gitlab-review` triggers (`review the MR !N`, `post review result to the MR`, `merge the request`) do need `glab` — don't type them if you don't use GitLab.

## If you want it gone entirely

This file is a deliberate pointer, not the spec. To remove the standalone skill completely, delete the `skills/review-branch/` directory and drop its row from the README — nothing depends on it.
