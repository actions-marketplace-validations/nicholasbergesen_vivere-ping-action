# vivere-ping-action

Tell a [Vivere](https://vivere.dev) monitor that a scheduled workflow ran, so that you are
alerted when one does not.

GitHub's own notifications tell you when a workflow **fails**. They cannot tell you when a
workflow **stops running**, because there is no run to notify you about. That happens more
than people expect:

- GitHub disables scheduled workflows in a public repository after **60 days of repository
  inactivity**, and scheduled runs do not count as activity. A repository that exists only to
  run a nightly job switches itself off roughly every two months.
- The `schedule` event only fires for the workflow file on the **default branch**, so a cron
  trigger added on a branch is never scheduled.
- Scheduled runs are queued, not guaranteed; at busy times, especially on the hour, they are
  delayed or dropped.
- For private repositories, exhausted Actions minutes or a spending limit stop runs starting.

This action makes the workflow check in when it succeeds. If the check-in does not arrive on
time, Vivere alerts you — by email, webhook, Slack, Discord, Telegram or ntfy.

## Usage

Create a heartbeat monitor at [vivere.dev](https://vivere.dev), set its period to your
schedule and its grace to how late a run may be, then put the ping URL in a repository
secret (Settings → Secrets and variables → Actions).

```yaml
name: Nightly export
on:
  schedule:
    - cron: '17 2 * * *'   # an odd minute: :00 is the busiest and the most delayed
  workflow_dispatch:

jobs:
  export:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: ./scripts/export.sh

      - uses: nicholasbergesen/vivere-ping-action@v1
        with:
          url: ${{ secrets.VIVERE_PING_URL }}
```

Because the ping step only runs if every step before it succeeded, a failure means no ping,
and the missed deadline raises the alert.

### Report failures immediately

Waiting for a missed deadline tells you up to one period late. Pass `job.status` on a step
that always runs and you are told at once:

```yaml
      - uses: nicholasbergesen/vivere-ping-action@v1
        if: always()
        with:
          url: ${{ secrets.VIVERE_PING_URL }}
          status: ${{ job.status }}
```

`success` stays a success; `failure` and `cancelled` report a failure.

### Record how long it took

Ping `start` first and Vivere records the duration of every run, so you can see the night a
job took four times as long before it becomes the night it did not finish.

```yaml
jobs:
  export:
    runs-on: ubuntu-latest
    steps:
      - uses: nicholasbergesen/vivere-ping-action@v1
        with:
          url: ${{ secrets.VIVERE_PING_URL }}
          status: start

      - uses: actions/checkout@v4
      - run: ./scripts/export.sh 2>&1 | tee /tmp/export.log

      - uses: nicholasbergesen/vivere-ping-action@v1
        if: always()
        with:
          url: ${{ secrets.VIVERE_PING_URL }}
          status: ${{ job.status }}
          log: ${{ github.run_id }} on ${{ github.sha }}
```

### Attach output to the run

Anything in `log` is stored with the run and shown beside the failure. Vivere keeps the last
64 KB.

```yaml
      - id: export
        run: ./scripts/export.sh > /tmp/out.txt 2>&1

      - if: always()
        run: echo "LOG<<EOF" >> $GITHUB_ENV && tail -c 8000 /tmp/out.txt >> $GITHUB_ENV && echo "EOF" >> $GITHUB_ENV

      - uses: nicholasbergesen/vivere-ping-action@v1
        if: always()
        with:
          url: ${{ secrets.VIVERE_PING_URL }}
          status: ${{ job.status }}
          log: ${{ env.LOG }}
```

## Inputs

| Input | Default | Description |
| --- | --- | --- |
| `url` | *required* | The monitor's ping URL, `https://vivere.dev/p/<monitor-id>`. Keep it in a secret. |
| `status` | `success` | `success`, `start`, `fail`, a number `0`–`255`, or a `job.status` value. |
| `log` | `''` | Text attached to the run; the last 64 KB is kept. |
| `timeout` | `10` | Seconds allowed for the whole call. |
| `retries` | `3` | Retries on a transient network failure. |
| `fail-on-error` | `false` | Whether a failed ping fails the step. |

### Output

| Output | Description |
| --- | --- |
| `pinged` | `true` when the monitor accepted the ping. |

## Why `fail-on-error` is off by default

A monitoring call is not worth breaking a build over, and a ping that never arrives raises
an alert by itself — which is the outcome you wanted from the failure anyway. Turn it on if
you would rather find out about a broken ping URL immediately.

## Keep the URL in a secret

Anyone holding the ping URL can mark your monitor as healthy, which is exactly as bad as it
sounds: it would suppress the alert you set the monitor up for. This action never echoes the
URL, and GitHub redacts secrets from logs, but a URL written directly into a workflow file
in a public repository is public.

## It is not GitHub-specific

The whole integration is one HTTP call, so the same monitor works from cron, systemd,
Kubernetes, Windows Task Scheduler, n8n or anything else:

```bash
curl -fsS -m 10 --retry 3 -o /dev/null https://vivere.dev/p/<monitor-id>
```

## Links

- [Vivere](https://vivere.dev) — free for 10 monitors, no card
- [GitHub Actions guide](https://vivere.dev/docs/github-actions)
- [Why a scheduled workflow stops running](https://vivere.dev/answers/github-actions-schedule-stopped)
- [Choosing a grace period](https://vivere.dev/tools/grace-period)

## Licence

MIT. See [LICENSE](LICENSE).
