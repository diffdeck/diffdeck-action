# diffdeck-action

GitHub Action that uploads **Storybook builds** and **Playwright recordings** to
[DiffDeck](https://diffdeck.ai) for hosted visual review and screenshot diffs.

It is a **thin wrapper** around the [`@diffdeckai/cli`](https://www.npmjs.com/package/@diffdeckai/cli)
npm package: it fills branch/commit metadata from the GitHub context, runs the CLI
via `npx`, and surfaces the resulting DiffDeck URL as an action output. All upload
logic lives in the CLI — this action implements none of it.

## Usage

Add your DiffDeck **project token** (shown when you add a repository in DiffDeck) as
a repository secret named `DIFFDECK_TOKEN`, then reference the action with the
`command` you need.

### Upload a Storybook build

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
          node-version: 20
      - run: npm ci
      - run: npm run build-storybook -- -o storybook-static
      - uses: diffdeck/diffdeck-action@v1
        with:
          command: storybook
          token: ${{ secrets.DIFFDECK_TOKEN }}
          dir: storybook-static
```

### Upload Playwright recordings

```yaml
name: DiffDeck Recordings
on: [push, pull_request]

jobs:
  diffdeck:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      - run: npm ci
      - run: npx playwright test # produces recordings in ./recordings
      - uses: diffdeck/diffdeck-action@v1
        with:
          command: recordings
          token: ${{ secrets.DIFFDECK_TOKEN }}
          dir: recordings
```

### Using the URL output

```yaml
      - uses: diffdeck/diffdeck-action@v1
        id: diffdeck
        with:
          command: storybook
          token: ${{ secrets.DIFFDECK_TOKEN }}
          dir: storybook-static
      - run: echo "Review at ${{ steps.diffdeck.outputs.url }}"
```

## Inputs

| Input         | Required | Default                                   | Description                                                                                              |
| ------------- | -------- | ----------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| `command`     | yes      | —                                         | `storybook` or `recordings`.                                                                             |
| `token`       | yes      | —                                         | DiffDeck project token. Use a repository secret (e.g. `${{ secrets.DIFFDECK_TOKEN }}`).                  |
| `dir`         | no       | `storybook-static`                        | Directory to upload (Storybook static dir, or the recordings output dir).                                |
| `host`        | no       | _(CLI default — `https://diffdeck.ai`)_   | DiffDeck base URL. Set this only for self-hosted / non-default deployments.                              |
| `branch`      | no       | `github.head_ref \|\| github.ref_name`    | Branch the upload is for. Defaults to the PR source branch on `pull_request`, otherwise the ref name.    |
| `sha`         | no       | `github.sha`                              | Commit SHA the upload is for.                                                                            |
| `cli-version` | no       | `latest`                                  | Version (npm dist-tag or semver) of `@diffdeckai/cli` to run.                                              |

## Outputs

| Output | Description                                                              |
| ------ | ------------------------------------------------------------------------ |
| `url`  | URL of the uploaded build / recording set on DiffDeck (parsed from CLI). |

## How it works

The action runs (roughly):

```bash
DIFFDECK_TOKEN=<token> \
  npx @diffdeckai/cli@<cli-version> upload-storybook --dir <dir> --branch <branch> --sha <sha> [--host <host>]
# or, for command: recordings
DIFFDECK_TOKEN=<token> \
  npx @diffdeckai/cli@<cli-version> upload-recording  --dir <dir> --branch <branch> --sha <sha> [--host <host>]
```

The token is passed to the CLI via the `DIFFDECK_TOKEN` environment variable (never
on the command line); the CLI sends it to DiffDeck as the `X-UI-Review-Token`
request header. The action captures the CLI's output and exposes the review URL it
prints as the `url` output.

## License

[MIT](./LICENSE)
