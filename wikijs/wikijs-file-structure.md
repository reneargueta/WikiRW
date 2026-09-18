# File Structure in Wiki.js (Git Storage)

**Purpose:** Understand how Wiki.js translates a page's path into an actual file path within the Git repository, in order to plan the wiki's taxonomy (folders, subpages, hierarchy) before migrating content.

---

## Core rule

When Wiki.js uses Git as storage, every page is saved as a file whose path is exactly the **page's path + the content type's extension**:

```
{page.path}.{extension}
```

Example: a page with path `research-pocs/token-economics` and markdown content is saved as:
```
research-pocs/token-economics.md
```

There's no intermediate mapping layer or separate database tracking "where each page lives" — **the repository's folder structure IS the wiki's navigation hierarchy**.

---

## Wiki.js doesn't have "folders" in the traditional sense

An important detail about how Wiki.js works: **you never need to explicitly create a folder**. If you create a page with path `/universe/planets/earth`, Wiki.js automatically infers that the levels `/universe` and `/universe/planets` exist, without you having to manually create anything — the system derives them directly from the path.

This gives it flexibility, but it means the visual hierarchy (breadcrumbs, navigation tree) depends entirely on how you name your paths, not on a folder structure you manage separately.

---

## How to create a "parent page" with subpages underneath

This is the most important pattern for your taxonomy. If you want a section with:
- A landing/summary page (e.g. "Research & POCs")
- Several subpages underneath (e.g. "Token Economics," "Traceability")

The file structure looks like this:

```
research-pocs.md                    ← parent page (path: /research-pocs)
research-pocs/
  ├── token-economics.md            ← subpage (path: /research-pocs/token-economics)
  ├── traceability.md               ← subpage (path: /research-pocs/traceability)
  └── ai-assistant-overview.md      ← subpage (path: /research-pocs/ai-assistant-overview)
```

This **doesn't create a conflict** on the filesystem, because `research-pocs.md` (file) and `research-pocs/` (folder) are two distinct things — one has an extension, the other doesn't.

**Why you should always create the parent page:** if `research-pocs.md` doesn't exist, clicking the "research-pocs" breadcrumb from any subpage lands the user on a nonexistent/empty page. Wiki.js's official documentation explicitly recommends creating this "landing" page for exactly that reason.

---

## Language (locale) prefix

If a page is in the instance's **default** configured language, the path is saved as-is, with no prefix.

If a page is in a language **other than** the default, Wiki.js prepends the language code as a folder:

```
en/research-pocs/token-economics.md
```

For a wiki with content in a single language (the most likely case for the BAP), this has no practical effect — but it's relevant if bilingual content gets added in the future.

---

## Known limitation: subpage navigation in the sidebar

A problem consistently reported by Wiki.js users: the side navigation panel **doesn't automatically show subpages (children)** while you're on the parent page — it only shows "siblings" at the same level. You have to explicitly click to expand the tree and see the children.

**Why this matters for the evaluation:** this is exactly the kind of navigation friction a purely hierarchical system can create — worth keeping in mind if Wiki.js ends up being the final recommendation, since the "browsing" experience isn't as smooth as in tools like Confluence or BookStack.

---

## Known limitation: images and assets with relative paths

Another real, well-reported gotcha: **images don't support paths relative to the `.md` file's folder** the way you'd expect (GitHub/GitLab-style, where an image placed next to the markdown file just works).

- An image placed next to the `.md` file looks fine while working locally, but **breaks once pushed to the remote repository**.
- The only currently supported approach is uploading images to an assets folder at the **repo root level**, not alongside each page.

**Why this matters:** if migrated content includes diagrams, screenshots, or supporting images (like the ones already used in the GitHub sync troubleshooting guide), a centralized assets folder needs to be planned from the start, rather than assuming each section can have its own images sitting next to its pages.

---

## Practical summary for BAP's taxonomy

| You need | How to do it |
|---|---|
| A top-level section (e.g. "Onboarding") | Create a page with path `onboarding` |
| Content within that section | Create pages with path `onboarding/something` |
| Subpage breadcrumbs to work | Make sure a page exists at the parent path (`onboarding.md`) |
| Multi-level hierarchy | Simply use more path segments (`onboarding/pega/initial-setup`) — no need to "create folders" |
| Images/diagrams on pages | Upload them to an assets folder at the repo root, not alongside each `.md` file |
