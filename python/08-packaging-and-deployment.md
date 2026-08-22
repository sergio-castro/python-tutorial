# Packaging and Deployment

You have a project with a `src/` layout and a `[build-system]`. This chapter turns it into
an artifact other people can install, and automates publishing that artifact to a private
registry.

## The Two Artifacts

Building a Python package produces **two** files, and they are not competitors — you publish
both:

| Artifact | Extension | What it is |
|----------|-----------------|--------------------------------------------------|
| **Wheel** | `.whl` | A **built** distribution. Essentially a zip of the files as they should land in `site-packages`. |
| **sdist** | `.tar.gz` | A **source** distribution. Your source tree plus the metadata needed to build a wheel from it. |

Installing a wheel is unzip-and-copy: no build step, no compiler, nothing executed on the
user's machine. Installing an sdist means the user's machine has to *build* your package
first — slower, and it needs whatever your build requires.

So why ship the sdist at all? Because it's the fallback and the source of truth:

- If no wheel matches the user's platform or Python version, pip can still build from the sdist.
- Linux distributions, security auditors, and corporate review processes want the real source.
- Some environments forbid installing prebuilt binaries as policy.

**Rule: always publish both.** `uv build` does this by default and it costs you nothing.

## What About Eggs?

You may run into `.egg` files or `easy_install` in older material. **Eggs are dead** — this is
one of the rare cases in Python packaging with no tradeoff to weigh.

The egg was setuptools' own format, predating any standard. The wheel replaced it via
[PEP 427](https://peps.python.org/pep-0427/), became the standard, and setuptools eventually
removed the tooling that produced eggs. Modern pip does not install them. If a tutorial tells
you to run `easy_install` or build a `bdist_egg`, it is more than a decade out of date.

One point of confusion worth clearing up: a `*.egg-info/` directory in a project is **not** an
egg. It's build metadata that some tools still generate locally. Harmless, and safe to
gitignore.

## Building

```bash
uv build
```

```
Building source distribution...
Building wheel from source distribution...
Successfully built dist/my_lib-0.1.0.tar.gz
Successfully built dist/my_lib-0.1.0-py3-none-any.whl
```

Note the second line: uv builds the sdist first, then builds the wheel **from that sdist**
rather than from your working directory. That's deliberate — it proves the sdist is complete.
If you forgot to include a file, the wheel build fails here rather than after a user downloads
a broken package.

`uv build --wheel` or `--sdist` builds just one, but you rarely want that.

`dist/` should be in `.gitignore`. Build artifacts are derived from source; committing them
means two sources of truth.

## Anatomy of a Wheel

A wheel is a zip file, so you can just look inside:

```bash
unzip -l dist/my_lib-0.1.0-py3-none-any.whl
```

```
my_lib/__init__.py
my_lib/py.typed
my_lib-0.1.0.dist-info/METADATA
my_lib-0.1.0.dist-info/WHEEL
my_lib-0.1.0.dist-info/RECORD
```

Two things to notice:

- **`src/` is gone.** The wheel contains `my_lib/`, not `src/my_lib/`. This is the layout being
  stripped at build time, exactly as [Project Configuration](./03-project-configuration.md)
  described — the wheel holds the files as they will sit in `site-packages`.
- **`.dist-info/` is generated.** `METADATA` holds your `[project]` table rendered into the
  standard format (name, version, dependencies, extras). `RECORD` lists every file with a hash,
  so the installer knows precisely what to remove on uninstall.

Compare with the sdist, which keeps the source tree intact:

```
my_lib-0.1.0/pyproject.toml
my_lib-0.1.0/README.md
my_lib-0.1.0/src/my_lib/__init__.py
my_lib-0.1.0/PKG-INFO
```

### The Filename Is Structured Data

```
my_lib   -  0.1.0   -  py3    -  none  -  any      .whl
└─name─┘    └ver─┘     └py─┘     └abi┘    └platform┘
```

Installers parse this to decide whether a wheel is usable. `py3-none-any` means **pure
Python** — no compiled code, works on any interpreter and any operating system. One file
serves everybody.

A package with C extensions is different:

```
numpy-2.1.0-cp312-cp312-manylinux_2_17_x86_64.whl
```

That wheel works only on CPython 3.12, on 64-bit Linux. Such projects publish a matrix of
wheels — one per Python version × platform — and the sdist stops being a nicety and becomes
the fallback for anything they didn't precompile.

This is the one genuine fork in the road, and **your code decides it, not you**: pure Python
gets one universal wheel, compiled extensions get many. Everything in this tutorial is pure
Python.

## Versions Are Immutable

Before publishing, understand the rule that surprises people most: **you cannot re-upload a
version.** Once `0.1.0` exists in a registry, that filename is permanently taken. Fix a typo
and you must ship `0.1.1`. Deleting the old one doesn't free the name on most registries.

Two consequences:

- Never publish from a dirty working tree. What you upload is what users get, forever.
- Automate publishing off a **tag**, not off every commit — a tag is a deliberate act, and it
  maps one-to-one with a version.

The version lives in `pyproject.toml`:

```toml
[project]
name = "my-lib"
version = "0.1.0"
```

Keeping that in sync with your git tags by hand is a well-known source of mistakes. Plugins
like `hatch-vcs` derive the version from the tag instead, so the tag becomes the single source
of truth. Worth adopting once releases become routine; the manual version is fine to start.

## Where Packages Go

The public registry is **PyPI**. Most companies also run a private one — **JFrog Artifactory**,
Nexus, AWS CodeArtifact, Azure Artifacts, GitLab's package registry.

The good news is that they all speak the same protocol. Artifactory exposes a PyPI-compatible
endpoint, so `uv`, `pip`, and `twine` work against it unchanged. There is nothing
JFrog-specific to learn beyond two URLs:

| Purpose | URL shape |
|-----------|--------------------------------------------------------------|
| Uploading | `https://<host>/artifactory/api/pypi/<repo-key>` |
| Installing | `https://<host>/artifactory/api/pypi/<repo-key>/simple` |

They differ by that trailing `/simple`. Mixing them up is the single most common setup error —
and the resulting message is usually an unhelpful `405` or a hang.

Declare both once in `pyproject.toml`:

```toml
[[tool.uv.index]]
name = "jfrog"
url = "https://mycompany.jfrog.io/artifactory/api/pypi/pypi-local/simple"
publish-url = "https://mycompany.jfrog.io/artifactory/api/pypi/pypi-local"
```

Now `uv publish --index jfrog` uploads to `publish-url`, and uses `url` to check what's already
there so a re-run skips files it has published before. (Worth having: PyPI tolerates
re-uploading an identical file, but most other registries — Artifactory included — return an
error.)

## Publishing by Hand First

Always do one manual publish before automating. Debugging credentials and URLs is far easier
in your own terminal than in a CI log.

```bash
uv build
uv publish --index jfrog --username "$JFROG_USER" --password "$JFROG_TOKEN"
```

Use `--dry-run` to validate everything without uploading. Note "password" here means a JFrog
**identity token**, not your account password — see the secrets section below.

---

# Automating with GitHub Actions

The rest of this chapter assumes you have never used GitHub Actions.

## What It Is

GitHub Actions is CI/CD built into GitHub. You commit a YAML file describing what to run and
when; GitHub watches your repository and, when something matching happens, runs your commands
on a **fresh virtual machine** that it provides and then throws away.

If you know Jenkins or GitLab CI, it's that. The differences: the config lives in your repo
next to the code, and you don't maintain the machines.

The key mental model is **the machine starts empty**. It has an OS and some common tooling, but
it does not have your code and it does not have uv. Your first steps have to put them there.

## Where the File Goes

```
your-repo/
└── .github/
    └── workflows/
        └── publish.yml
```

The `.github/workflows/` path is mandatory — that's where GitHub looks. The filename is yours.
A repo can hold many workflow files; each is independent.

## The Workflow

```yaml
name: Publish to JFrog

on:
  push:
    tags:
      - "v*"

permissions:
  contents: read

jobs:
  publish:
    runs-on: ubuntu-latest

    steps:
      - name: Check out the repository
        uses: actions/checkout@v7

      - name: Install uv
        uses: astral-sh/setup-uv@v9

      - name: Build the sdist and wheel
        run: uv build

      - name: Publish to JFrog
        run: uv publish --index jfrog
        env:
          UV_PUBLISH_USERNAME: ${{ secrets.JFROG_USERNAME }}
          UV_PUBLISH_PASSWORD: ${{ secrets.JFROG_TOKEN }}
```

That's the whole thing. Now every line.

### `name: Publish to JFrog`

A human label. It's what appears in your repository's **Actions** tab. Cosmetic — but with
several workflows, meaningful names save you a lot of clicking.

### `on:` — the trigger

```yaml
on:
  push:
    tags:
      - "v*"
```

**`on`** answers *when should this run?* Here: whenever a **tag** starting with `v` is pushed —
`v0.1.0`, `v1.2.3`.

Two decisions are packed into that:

- **Tags, not every push.** Because versions are immutable, publishing on every commit would
  fail on the second commit — the version hasn't changed, so the upload is a duplicate. A tag
  is a deliberate "this is a release."
- **`v*` is a glob**, not a regex. `v*` matches anything starting with `v`. Quote it — a bare
  `*` at the start of a YAML value is a syntax error.

Other common triggers you'll meet: `on: push: branches: [main]`, `on: pull_request`, and
`on: workflow_dispatch` (adds a "Run workflow" button for manual runs).

### `permissions: contents: read`

Each run gets an automatic credential for talking to GitHub. By default it can be broader than
needed. This line says the workflow may only *read* the repository — it cannot push commits,
open pull requests, or publish releases.

Not strictly required, but it's one line and it means a compromised dependency in your build
can't rewrite your repo. Set the minimum a workflow actually needs.

### `jobs:` and `publish:`

```yaml
jobs:
  publish:
```

A workflow contains one or more **jobs**. Each job gets its **own fresh machine**, and jobs run
**in parallel** unless you declare a dependency between them (see hardening, below).

`publish` is the job's ID — you choose it. It matters when another job needs to reference this
one.

### `runs-on: ubuntu-latest`

Which machine to rent. `ubuntu-latest` is the default choice: fastest to start and the cheapest
in billed minutes. `windows-latest` and `macos-latest` exist if you need them; a pure-Python
package does not.

Remember: **fresh and empty every run.** Nothing carries over from the last run.

### `steps:` — `uses` vs `run`

Steps execute top to bottom on the same machine. Each step is one of two kinds, and this
distinction is the thing to internalize:

| Key | Meaning |
|-------|-----------------------------------------------------------------------|
| `uses` | Run a **prebuilt action** — a reusable step someone else published. |
| `run` | Run a **shell command**, exactly as you would in a terminal. |

`name:` on a step is optional and purely for the log.

### Step 1 — `actions/checkout@v7`

```yaml
- name: Check out the repository
  uses: actions/checkout@v7
```

The machine starts without your code. This action clones the repository into it. **Nearly every
workflow starts with this**, and omitting it produces a baffling "no such file or directory"
on the next step.

`actions/checkout` is the repository path — GitHub's own, in the `actions` organization. `@v7`
pins the major version: you get v7's bug fixes automatically, but v8's breaking changes never
arrive unannounced.

### Step 2 — `astral-sh/setup-uv@v9`

```yaml
- name: Install uv
  uses: astral-sh/setup-uv@v9
```

Installs uv on the runner and caches it between runs. Published by Astral, who make uv.

You don't need a separate step to install Python — uv downloads the interpreter your
`.python-version` and `requires-python` call for.

### Step 3 — `uv build`

```yaml
- name: Build the sdist and wheel
  run: uv build
```

A plain shell command, hence `run`. It produces `dist/` on the runner, exactly as it does
locally. `dist/` disappears with the machine when the run ends, which is fine — the next step
uploads it.

### Step 4 — `uv publish`

```yaml
- name: Publish to JFrog
  run: uv publish --index jfrog
  env:
    UV_PUBLISH_USERNAME: ${{ secrets.JFROG_USERNAME }}
    UV_PUBLISH_PASSWORD: ${{ secrets.JFROG_TOKEN }}
```

`--index jfrog` refers to the `[[tool.uv.index]]` block in `pyproject.toml` — the URLs live in
the repo, so the workflow stays short and the same command works locally.

**`env:`** sets environment variables for this step only. uv reads `UV_PUBLISH_USERNAME` and
`UV_PUBLISH_PASSWORD` automatically, which is why no `--username` flag appears. Passing
credentials as environment variables rather than command-line flags also keeps them out of
process listings and logs.

**`${{ secrets.X }}`** is GitHub's interpolation syntax, pulling from encrypted repository
secrets. GitHub masks these values in logs — printing one shows `***`.

## Setting Up the Secrets

The workflow refers to two secrets that don't exist yet.

**1. Create a JFrog identity token.** In Artifactory: your profile → **Generate an Identity
Token**. Use a token, never your account password — tokens are scopable and revocable
individually. It needs *deploy* permission on the target repository. Copy it; it is shown once.

**2. Add both secrets to GitHub.** In your repository: **Settings** → **Secrets and variables**
→ **Actions** → **New repository secret**. Add:

| Name | Value |
|------------------|--------------------------------------------|
| `JFROG_USERNAME` | Your Artifactory username or email |
| `JFROG_TOKEN` | The identity token from step 1 |

The names must match the `${{ secrets.… }}` references exactly. Secrets are write-only — you
can overwrite or delete one, but never read it back.

**Never put a token in the YAML file.** Workflow files are committed, and anyone with read
access to the repo — or to its history, after you delete it — can see them.

## Running It

```bash
git tag v0.1.0
git push origin v0.1.0
```

`git push` alone does **not** push tags; you have to name the tag (or use `--tags`). This trips
up almost everyone the first time: the tag exists locally, nothing happens on GitHub, and the
workflow appears broken.

Open the **Actions** tab and you'll see the run appear. Click into it to expand each step's log.

## When It Fails

| Symptom | Cause |
|--------------------------------------------|--------------------------------------------------------------------------|
| Workflow never appears in the Actions tab | The tag didn't reach GitHub (`git push origin v0.1.0`), the pattern doesn't match, or the workflow file isn't on the default branch yet |
| `401 Unauthorized` | Wrong username or token — check for a trailing newline when you pasted the secret |
| `403 Forbidden` | Credentials are valid but the token lacks *deploy* permission on that repo |
| `409 Conflict` / "already exists" | That version is already published. Bump the version and tag again. |
| `405 Method Not Allowed`, or a hang | Publishing to the `/simple` URL. Upload uses the URL **without** `/simple`. |
| `No such file or directory: dist/` | The `uv build` step is missing, or `actions/checkout` is |

**GitHub Actions only reads the workflow file from the commit being built.** While iterating,
your changes must be pushed before they take effect — editing locally and re-running the old
run does nothing.

## Hardening It

The workflow above is deliberately minimal. Three worthwhile additions:

**1. Test before publishing.** Add a second job and make publishing depend on it. `needs: test`
means `publish` waits, and is skipped entirely if `test` fails:

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: astral-sh/setup-uv@v9
      - run: uv sync
      - run: uv run pytest

  publish:
    needs: test
    runs-on: ubuntu-latest
    steps:
      # ... as before
```

**2. Verify the tag matches the version.** Nothing stops you tagging `v0.2.0` while
`pyproject.toml` still says `0.1.0`. This step fails the run instead of publishing the wrong
number:

```yaml
- name: Check tag matches project version
  run: |
    TAG="${GITHUB_REF_NAME#v}"
    VERSION=$(uv version --short)
    test "$TAG" = "$VERSION" || {
      echo "Tag $TAG does not match version $VERSION"; exit 1;
    }
```

`GITHUB_REF_NAME` is one of many variables GitHub injects — here, the tag name. `#v` is shell
prefix-stripping, turning `v0.1.0` into `0.1.0`.

**3. Drop the long-lived token.** Artifactory supports OIDC, letting a workflow exchange a
short-lived GitHub-issued identity for an Artifactory token at run time. Nothing durable is
stored in your repository secrets. More setup, and the right end state for a production
pipeline.

## Installing Your Package Back

Publishing is only half of it. To consume from the same registry, the `[[tool.uv.index]]` block
already declares where to look:

```bash
export UV_INDEX_JFROG_USERNAME="your-username"
export UV_INDEX_JFROG_PASSWORD="your-token"

uv add my-lib
```

The environment variable names are built from the index name — index `jfrog` becomes
`UV_INDEX_JFROG_*`. Credentials stay out of `pyproject.toml`, which is committed.

By default uv searches your extra indexes **in addition to** PyPI. If a package must come only
from your private registry, mark the index `explicit = true` and request it by name — this
avoids **dependency confusion**, where an attacker publishes a package to PyPI under your
internal name and hopes the resolver picks theirs.

## Cheat Sheet

```bash
# --- Build ---
uv build                              # sdist + wheel into dist/
uv build --wheel                      # wheel only
unzip -l dist/*.whl                   # inspect what you actually shipped

# --- Publish ---
uv publish --index jfrog --dry-run    # validate without uploading
uv publish --index jfrog              # credentials via UV_PUBLISH_USERNAME/PASSWORD

# --- Release ---
git tag v0.1.0
git push origin v0.1.0                # plain `git push` does NOT send tags

# --- Consume ---
export UV_INDEX_JFROG_USERNAME=... UV_INDEX_JFROG_PASSWORD=...
uv add my-lib
```

```toml
# pyproject.toml — one block, both URLs
[[tool.uv.index]]
name = "jfrog"
url = "https://mycompany.jfrog.io/artifactory/api/pypi/pypi-local/simple"   # install
publish-url = "https://mycompany.jfrog.io/artifactory/api/pypi/pypi-local"  # upload
```

Three things to carry away: **ship both the wheel and the sdist**; **a published version can
never be changed**, so release from tags; and in a workflow, **the machine starts empty** —
check out the code and install your tools before anything else.

---

Previous: [Linting and Formatting](./07-linting-and-formatting.md) |
Back to [Index](./tutorial.md)
