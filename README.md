# diffdeck-action

GitHub Action that renders + uploads **Storybook screenshots** and uploads
**Playwright recordings** to [DiffDeck](https://diffdeck.ai) for hosted visual
review and screenshot diffs.

It is a **thin wrapper** around the [`@diffdeckai/cli`](https://www.npmjs.com/package/@diffdeckai/cli)
npm package: it fills branch/commit metadata from the GitHub context, runs the CLI
via `npx`, and surfaces the resulting DiffDeck URL as an action output. All upload
logic lives in the CLI — this action implements none of it.

## Modes

The `command` input selects what to do (default **`auto`**):

| `command`    | CLI command            | What it does                                                                                     |
| ------------ | ---------------------- | ------------------------------------------------------------------------------------------------ |
| `auto`       | _(detected)_           | Runs every applicable mode: a **recording** when `video` is set, and a **screenshot** pass when the Storybook `dir` exists. |
| `screenshot` | `screenshot-storybook` | Renders every story **in CI** (six variants) + render-check, then uploads the screenshots + build. A story that fails to render fails the job. |
| `storybook`  | `upload-storybook`     | Uploads the built Storybook only; the server renders the screenshots.                            |
| `recording`  | `upload-recording`     | Uploads a single Playwright recording (video + test metadata).                                   |

> **Screenshot mode needs Playwright** (with browsers) in your project. Add it as a
> devDependency (`npm i -D playwright`); the action runs `playwright install --with-deps chromium`
> for you unless you set `install-browsers: false`.

## Usage

Add your DiffDeck **project token** (shown when you add a repository in DiffDeck) as
a repository secret named `DIFFDECK_TOKEN`.

### Screenshot a Storybook in CI (recommended)

```yaml
name: DiffDeck Storybook
on: [push, pull_request]

jobs:
  diffdeck:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0 # so commit metadata is available
      - uses: actions/setup-node@v4
        with:
          node-version: 22
      - run: npm ci
      - run: npm run build-storybook -- --output-dir storybook-static
      - uses: diffdeck/diffdeck-action@v1
        with:
          # command: auto  # the default — a Storybook dir present ⇒ screenshot mode
          token: ${{ secrets.DIFFDECK_TOKEN }}
          dir: storybook-static
```

### Upload a Storybook build (server renders)

```yaml
      - uses: diffdeck/diffdeck-action@v1
        with:
          command: storybook
          token: ${{ secrets.DIFFDECK_TOKEN }}
          dir: storybook-static
```

### Upload a Playwright recording

```yaml
      - uses: diffdeck/diffdeck-action@v1
        with:
          command: recording
          token: ${{ secrets.DIFFDECK_TOKEN }}
          video: test-results/home.webm
          test: "Home page renders"
          file: "tests/home.spec.ts"
          status: passed
```

> For a whole Playwright run, the [`@diffdeckai/playwright-reporter`](https://www.npmjs.com/package/@diffdeckai/playwright-reporter)
> is the better fit — it uploads every recording with metadata automatically. Use
> this action's `recording` mode for one-off single-video uploads.

### Using the URL output

```yaml
      - uses: diffdeck/diffdeck-action@v1
        id: diffdeck
        with:
          token: ${{ secrets.DIFFDECK_TOKEN }}
          dir: storybook-static
      - run: echo "Review at ${{ steps.diffdeck.outputs.url }}"
```

## Inputs

| Input              | Required | Default                                | Description                                                                                     |
| ------------------ | -------- | -------------------------------------- | ----------------------------------------------------------------------------------------------- |
| `command`          | no       | `auto`                                 | `auto`, `screenshot`, `storybook`, or `recording`.                                              |
| `token`            | yes      | —                                      | DiffDeck project token. Use a repository secret (e.g. `${{ secrets.DIFFDECK_TOKEN }}`).         |
| `dir`              | no       | `storybook-static`                     | Built Storybook static directory (for `screenshot`/`storybook`).                                |
| `video`            | no       | —                                      | Recorded Playwright video file (for `recording`).                                               |
| `host`             | no       | _(CLI default — `https://diffdeck.ai`)_| DiffDeck base URL. Set only for self-hosted / non-default deployments.                          |
| `branch`           | no       | `github.head_ref \|\| github.ref_name` | Branch the upload is for.                                                                        |
| `commit`           | no       | `github.sha`                           | Commit SHA the upload is for.                                                                    |
| `message`          | no       | _(commit subject line)_                | Commit message for Storybook builds.                                                             |
| `install-browsers` | no       | `true`                                 | In `screenshot` mode, run `playwright install --with-deps chromium` first.                       |
| `test`             | no       | —                                      | Recording: test title.                                                                           |
| `file`             | no       | —                                      | Recording: test file path.                                                                       |
| `test-id`          | no       | —                                      | Recording: stable test identifier.                                                              |
| `status`           | no       | —                                      | Recording: test status (`passed`, `failed`, …).                                                 |
| `duration`         | no       | —                                      | Recording: test duration in milliseconds.                                                        |
| `retries`          | no       | —                                      | Recording: number of retries.                                                                    |
| `metadata`         | no       | —                                      | Recording: extra metadata as a JSON string.                                                      |
| `cli-version`      | no       | `latest`                               | Version (npm dist-tag or semver) of `@diffdeckai/cli` to run.                                    |

## Outputs

| Output | Description                                                          |
| ------ | ------------------------------------------------------------------- |
| `url`  | URL of the uploaded build / recording on DiffDeck (parsed from CLI). |

## How it works

The action runs (roughly):

```bash
# screenshot (the auto default when a Storybook dir is present)
DIFFDECK_TOKEN=<token> \
  npx @diffdeckai/cli@<cli-version> screenshot-storybook --dir <dir> --branch <branch> --commit <commit> [--message <msg>] [--host <host>]

# storybook (server renders)
DIFFDECK_TOKEN=<token> \
  npx @diffdeckai/cli@<cli-version> upload-storybook --dir <dir> --branch <branch> --commit <commit> [--message <msg>] [--host <host>]

# recording
DIFFDECK_TOKEN=<token> \
  npx @diffdeckai/cli@<cli-version> upload-recording --video <video> --branch <branch> --commit <commit> [metadata…] [--host <host>]
```

The token is passed to the CLI via the `DIFFDECK_TOKEN` environment variable (never
on the command line); the CLI sends it to DiffDeck as the `X-UI-Review-Token`
request header. The action captures the CLI's output and exposes the review URL it
prints as the `url` output.

## License

[MIT](./LICENSE)
