# PR Size Helper action

This action adds [size labels](https://github.com/kubernetes/kubernetes/labels?q=size) to pull requests. If a pull requests is above a configured complexity threshold (calculated by default by summing lines of additions and deletions), the action will prompt the PR author for more context to help explain the size of the PR. If the author chooses to give additional context, the reason will be tracked along with all others in a digest issue.

The end goal is to proactively capture reasons why pull requests are above a certain number of changes and to index all of those reasons in one easy to find place.

## Demo

**1. A contributor creates a new PR:**

![a large pr is created](https://user-images.githubusercontent.com/1746081/112671818-e7432600-8e1f-11eb-8ca4-d6849eb77b14.png)


**2. pr-size-helper-action labels it with a (configurable) `size/` label:**

![pr is labeled with size label](https://user-images.githubusercontent.com/1746081/112671828-ee6a3400-8e1f-11eb-9225-e3021fc31896.png)

**3. If the PR crosses a (configurable) change size threshold the PR creator is prompted to provide more context:**

![action prompts author for reason comment](https://user-images.githubusercontent.com/1746081/112671845-f629d880-8e1f-11eb-9bd7-487b682681c2.png)

**4. When a `!reason` comment is provided, pr-size-helper-action captures the comment in a digest issue (with configurable destination):**

![reason comment is provided and captured](https://user-images.githubusercontent.com/1746081/112671861-fb872300-8e1f-11eb-9ec8-6b720ac99a90.png)

**5. The comment is added to the digest issue along with all other `!reason` comments:**

![digest issue displays all reason comments](https://user-images.githubusercontent.com/1746081/112671878-ff1aaa00-8e1f-11eb-884f-5e6d1f867809.png)

## Usage

Create two workflow files:

`.github/workflows/apply-pr-size-label.yml`

```
name: Apply PR size label

on: pull_request

jobs:
  apply_pr_size_label:
    runs-on: ubuntu-latest
    steps:
      - uses: levindixon/pr-size-helper-action@v1.5.0
        env:
          GITHUB_TOKEN: "${{ secrets.GITHUB_TOKEN }}"

```

`.github/workflows/track-large-pr-reasons.yml`

```
name: Track large PR reasons

on: issue_comment

jobs:
  track_large_pr_reasons:
    if: ${{ github.event.issue.pull_request && contains(github.event.comment.body, '!reason') }}
    runs-on: ubuntu-latest
    steps:
      - uses: levindixon/pr-size-helper-action@v1.5.0
        env:
          GITHUB_TOKEN: "${{ secrets.GITHUB_TOKEN }}"

```

## Configuration

The following environment variables are supported:

- `IGNORED`: A list of [glob expressions](http://man7.org/linux/man-pages/man7/glob.7.html)
  separated by newlines. Files matching these expressions will not count when
  calculating the complexity of the pull request. Lines starting with `#` are
  ignored and files matching lines starting with `!` are always included.
- `PROMPT_THRESHOLD`: Pull requests created with a complexity score greater or equal to this value will trigger a friendly message prompting the pull request author to provide a reason for the size of the pull request. Defaults to 500.
- `S` | `M` | `L` | `XL` | `XXL`: Setting one, some, or all of these will change the pull request size labelling. Pull requests with a complexity score between 0 and `S` will be labeled as `size/XS`, PRs with a size between `S` and `M` will be labeled as `S` and so on. Defaults:
  - `S`: 10
  - `M`: 30
  - `L`: 100
  - `XL`: 500
  - `XXL`: 1000
- `DIGEST_ISSUE_REPO`: The location of the digest issue, by default the digest issue will be created and updated in the repo where the action is configured. If you would like the digest issue to be created and updated in a repo outside of where the action is configured, set this to the url of the repo (e.g. "https://github.com/octokit/core.js") **This requires ACCESS_TOKEN to be configured.**
- `ACCESS_TOKEN`: This is a [GitHub personal access token](https://docs.github.com/en/github/authenticating-to-github/creating-a-personal-access-token) with the `repo` scope, stored as a [secret](https://docs.github.com/en/actions/reference/encrypted-secrets#creating-encrypted-secrets-for-a-repository) in the repo where this action is configured.
- `TEAMS`: This a space-delimited string of [team](https://docs.github.com/en/organizations/organizing-members-into-teams/about-teams) slugs, that restricts
this workflow from running any time that the PR author is not a member of one of
the specified teams. The owning organization for each team is assumed to be the
the owner of the repository that acts as the base of the newly opened pull request.
- `SCORING_STRATEGY`: An optional space-delimited list of strategies to use when calculating the complexity of a PR. By default, the complexity score is calculated by assigning 1 point to each line added and 1 point to each line removed, excluding any files specified in `IGNORED` as well as any whitespace lines and comment lines (limited language support for now, but more coming). The optional strategies listed below modify this behavior rather than replace it:
  - `tests-are-less-complex`: This strategy subtracts 0.5 points for each line change in test files.
  - `single-words-are-less-complex`: This strategy subtracts 0.5 points for each line change where the content of the line being changed is a single word. This is based on the assumption that if a line contains nothing but a single word (optionally surrounded by quotes or simple punctuation), it contributes less to overall complexity than other kinds of changes, and may in fact be formatted this way to improve readability/comprehension.

You can configure the environment variables in the `apply-pr-size-label.yml` workflow file like this:

```yaml
env:
  GITHUB_TOKEN: "${{ secrets.GITHUB_TOKEN }}"
  IGNORED: ".*\n!.gitignore\nyarn.lock\ngenerated/**"
  PROMPT_THRESHOLD: 500
  S: 10
  M: 30
  L: 100
  XL: 500
  XXL: 1000
  DIGEST_ISSUE_REPO: "https://github.com/octokit/core.js"
  ACCESS_TOKEN: "${{ secrets.ACCESS_TOKEN }}"
```

### Example configuration for action that publishes it's digest issue outside of the repo where it's configured

In this example we have two repos, one where the action will run (`levindixon/demo-project`) and another where the action will publish and maintain the digest issue (`levindixon/demo-project-two`).

A [personal access token](https://docs.github.com/en/github/authenticating-to-github/creating-a-personal-access-token) is generated with the `repo` scope and stored in `levindixon/demo-project` as a [secret](https://docs.github.com/en/actions/reference/encrypted-secrets#creating-encrypted-secrets-for-a-repository) named `ACCESS_TOKEN`

Two workflow files are created in `levindixon/demo-project`:

`.github/workflows/apply-pr-size-label.yml`

```
name: Apply PR size label

on: pull_request

jobs:
  apply_pr_size_label:
    runs-on: ubuntu-latest
    steps:
      - uses: levindixon/pr-size-helper-action@v1.5.0
        env:
          GITHUB_TOKEN: "${{ secrets.GITHUB_TOKEN }}"

```

`.github/workflows/track-large-pr-reasons.yml`

```
name: Track large PR reasons

on: issue_comment

jobs:
  track_large_pr_reasons:
    if: ${{ github.event.issue.pull_request && contains(github.event.comment.body, '!reason') }}
    runs-on: ubuntu-latest
    steps:
      - uses: levindixon/pr-size-helper-action@v1.5.0
        env:
          GITHUB_TOKEN: "${{ secrets.GITHUB_TOKEN }}"
          ACCESS_TOKEN: "${{ secrets.ACCESS_TOKEN }}"
          DIGEST_ISSUE_REPO: "https://github.com/levindixon/demo-project-two"

```

The result is PRs will be labeled and watched for `!reason` comments in `levindixon/demo-project`, however `!reason` comments will be tracked in a digest issue located in `levindixon/demo-project-two`

This configuration allows you to configure the action in any number of repositories and maintain a single digest issue for any/all of them!

## Acknowledgments

- 📝 Repo templated using [`actions/javascript-action`](https://github.com/actions/javascript-action) 📝

- ✨ Guided by the [Creating a JavaScript action](https://docs.github.com/en/actions/creating-actions/creating-a-javascript-action) guide ✨

- 🙇‍♂️ Ignore and labeling functionality forked from [`pascalgn/size-label-action`](https://github.com/pascalgn/size-label-action) 🙇‍♂️

- 💬 Prompt inspiration from [`CodelyTV/pr-size-labeler`](https://github.com/CodelyTV/pr-size-labeler) 💬

- 🏷 Size labels borrowed from [`kubernetes/kubernetes`](https://github.com/kubernetes/kubernetes/labels?q=size) 🏷

## License

[MIT](LICENSE)
