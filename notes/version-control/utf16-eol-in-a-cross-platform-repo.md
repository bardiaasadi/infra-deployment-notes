# UTF-16 Files, Line Endings, and Why Git Sometimes Treats Text Like a Binary

## The situation
In cross-platform repos, line endings are a constant background concern:

- Windows typically uses **CRLF**
- macOS/Linux typically use **LF**

Most version control systems handle this well by keeping a normalized representation and converting at checkout time.

The problem is: **Git does not reliably treat UTF-16 files as text**, and that changes everything.

In our repo, UTF-16 files were a small minority (hundreds out of tens of thousands), but they were high-impact because they were real, active configuration/resource files used by Windows tooling and builds.

## The core issue
Git often classifies UTF-16 files as **binary**.

That has two big implications:

1) **No meaningful diffs/merges**
Git can’t do normal line-based diffs and merges on files it believes are binary.

2) **No line-ending normalization**
Git’s EOL conversion behaves like a text feature. If a file is treated as binary, it can get “stuck” with whatever line endings it originally had—creating cross-platform pain.

### The dangerous trap
A naive fix is: “Tell Git it’s text.”

But if Git doesn’t understand the encoding, simply forcing `text` can lead to:
- incorrect conversions
- broken working copies
- in worst cases, corrupted content

So the fix has to be explicit and unambiguous.

## The solution
Use `.gitattributes` to explicitly define:

- the file is text (so it can be diffed / normalized)
- what encoding Git should use for the working tree
- what EOL should be checked out

Example (pattern-based):

```gitattributes
*.rc text diff working-tree-encoding=UTF-16LE-BOM eol=crlf
````

Or file-specific (path sanitized):

```gitattributes
path/to/file.rc text diff working-tree-encoding=UTF-16LE-BOM eol=crlf
```

This tells Git:

* “Do not guess. I’m telling you what this is.”
* “Treat it as text for diff.”
* “When writing to disk, use UTF-16LE with BOM and CRLF.”

## The second half: you must re-check-in / renormalize

Adding `.gitattributes` changes how Git *should* interpret the file, but existing history/index state may still reflect the old “binary” classification.

In practice, the fix required re-adding/recommitting affected files so Git re-indexed them under the new rules.

A common way to do that is renormalization:

```bash
git add --renormalize .
git commit -m "Normalize UTF-16 working-tree encoding and EOL via .gitattributes"
```

## Operational policy (how to keep it healthy)

Once the repo is in a healthy state, the biggest risk is **new UTF-16 files silently entering the codebase** and reintroducing the same class of issues.

A practical policy:

* detect new UTF-16 additions (automated check)
* either convert them to a more Git-friendly encoding, or
* add explicit `.gitattributes` entries and recommit under correct rules

## Why this is a “core engineering” story

Most teams never hit this problem because they don’t have:

* cross-platform dev in one repo
* UTF-16 resource/config files in active use
* strict build expectations that break on subtle EOL/encoding differences

When you do hit it, it’s a perfect example of platform engineering:

> reducing a class of failures by making the system explicit instead of relying on defaults.

