I'll document the complete GitHub pull request review setup, covering both personal repositories and GitHub Organization repositories.

The documentation will include:

1. CODEOWNERS configuration.
2. Branch protection rules.
3. Required and optional reviewers.
4. GitHub Actions automation for optional reviewers.
5. Organization-level Rulesets and Required Reviewer Rules.
6. End-to-end workflow, examples, troubleshooting, and recommended configuration.

# GitHub Pull Request Review Management

Technical documentation · GitHub · Repository governance and branch protection

## 1. Objective

The objective is to establish a standardized pull request (PR) review process that ensures:

- Required reviewers must approve changes before merging.
- Optional reviewers are automatically requested but their approval is not mandatory.
- Different teams can review specific files or directories.
- Direct pushes to protected branches are restricted.
- Review policies are consistently enforced across repositories.

## 2. Understanding GitHub review mechanisms

GitHub provides three main mechanisms for managing PR reviews.

| Feature                 | Purpose                                                                     |
| ----------------------- | --------------------------------------------------------------------------- |
| CODEOWNERS              | Automatically requests reviews from designated users or teams.              |
| Branch Protection Rules | Enforces approval requirements before merging.                              |
| GitHub Actions          | Automates reviewer assignment and other PR activities.                      |
| Organization Rulesets   | Centralizes and enforces repository review policies across an organization. |

Important distinction: CODEOWNERS requests reviewers. Branch protection or Rulesets enforce mandatory approval requirements. GitHub Actions can automate optional reviewer assignment.

## 3. Configure CODEOWNERS

### 3.1 Create the CODEOWNERS file

Create the following file in your repository:

```
.github/CODEOWNERS
```

GitHub also supports placing it in the repository root or `docs/` directory. The `.github/` directory is recommended.

### 3.2 Configure a mandatory reviewer

Example:

```
# Mandatory reviewer for all repository changes
* @user1
```

Here:

- `*` matches all files.
- `@user1` is automatically requested to review matching PRs.
- The user must have Write access to the repository.

### 3.3 Configure different reviewers for different directories

```
# Default mandatory reviewer
* @user1

# Terraform files
/terraform/ @devops-user

# Kubernetes manifests
/kubernetes/ @platform-user

# Application source code
/src/ @development-user

# GitHub Actions workflows
/.github/ @devops-user
```

Note: CODEOWNERS uses the last matching pattern for a file. More specific rules can override the default ownership rule.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://docs.github.com\&sz=32)

GitHub Docs+1



### 3.4 Important CODEOWNERS considerations

- The CODEOWNERS file must exist in the PR's base branch.
- Users must have Write access to the repository.
- Team owners must have explicit repository access.
- Draft PRs do not automatically request code-owner reviews until marked ready for review.
- If multiple owners are defined on the same line, approval from any one owner satisfies the code-owner requirement when that requirement is enabled.

For example:

```
* @user1 @user2
```

This does not mean both users must approve. If you need `user1` to be mandatory and `user2` optional, use separate configurations.

## 4. Configure Branch Protection Rules

Branch protection ensures that developers cannot merge changes without satisfying the required review conditions.

### 4.1 Navigate to branch protection

1. Open the GitHub repository.
2. Navigate to Settings.
3. Select Branches.
4. Under Branch protection rules, click Add branch protection rule.
5. Enter the branch name, such as `main`.

### 4.2 Configure protection settings

Recommended configuration:

| Setting                                             | Configuration |
| --------------------------------------------------- | ------------- |
| Require a pull request before merging               | Enabled       |
| Require approvals                                   | Enabled       |
| Required approvals                                  | 1             |
| Dismiss stale pull request approvals                | Enabled       |
| Require review from Code Owners                     | Enabled       |
| Require approval of the most recent reviewable push | Enabled       |
| Allow force pushes                                  | Disabled      |
| Allow deletions                                     | Disabled      |

The exact settings can vary according to your organization's security requirements.

### 4.3 What these settings enforce

- Developers must create PRs instead of directly pushing to the protected branch.
- At least one eligible reviewer must approve.
- The designated code owner must approve changes to owned files.
- New changes can invalidate previous approvals if stale-review dismissal is enabled.

Official documentation: [GitHub Protected Branches](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches?utm_source=chatgpt.com).

## 5. Configure optional reviewers using GitHub Actions

For personal repositories, GitHub Actions can automatically request additional reviewers without making their approval mandatory.

### 5.1 Expected requirement

Suppose you have two reviewers:

| User  | Reviewer type | Approval      |
| ----- | ------------- | ------------- |
| user1 | Required      | Mandatory     |
| user2 | Optional      | Not mandatory |

Expected behavior:

| User1    | User2    | Merge allowed? |
| -------- | -------- | -------------- |
| Approved | Pending  | Yes            |
| Approved | Approved | Yes            |
| Pending  | Approved | No             |
| Pending  | Pending  | No             |

### 5.2 Configure CODEOWNERS

File: `.github/CODEOWNERS`

```
# Mandatory reviewer
* @user1
```

### 5.3 Create the GitHub Actions workflow

Create:

```
.github/workflows/auto-assign-reviewer.yml
```

Add the following configuration:

```
name: Auto Assign Optional Reviewer

on:
  pull_request:
    types:
      - opened
      - reopened
      - ready_for_review

permissions:
  contents: read
  pull-requests: write

jobs:
  assign-reviewer:
    runs-on: ubuntu-latest

    steps:
      - name: Request Optional Reviewer
        uses: actions/github-script@v7
        with:
          script: |
            const reviewer = 'user2';

            const { owner, repo } = context.repo;
            const pull_number = context.issue.number;

            const { data: pr } =
              await github.rest.pulls.get({
                owner,
                repo,
                pull_number
              });

            // Avoid requesting the PR author
            if (
              pr.user.login.toLowerCase() ===
              reviewer.toLowerCase()
            ) {
              core.info('Reviewer is the PR author. Skipping.');
              return;
            }

            // Check existing review requests
            const alreadyRequested =
              pr.requested_reviewers.some(
                user =>
                  user.login.toLowerCase() ===
                  reviewer.toLowerCase()
              );

            if (alreadyRequested) {
              core.info('Reviewer already requested.');
              return;
            }

            // Request optional reviewer
            await github.rest.pulls.requestReviewers({
              owner,
              repo,
              pull_number,
              reviewers: [reviewer]
            });

            core.info(
              `Requested optional review from ${reviewer}`
            );
```

Replace `user2` with the actual GitHub username.

### 5.4 Configure workflow permissions

The workflow requires:

```
permissions:
  contents: read
  pull-requests: write
```

This gives the workflow permission to read repository contents and request PR reviews.

The optional reviewer must have access to the repository and be eligible to receive a review request.

### 5.5 How the workflow works

1. Developer creates a PR.
2. GitHub Actions starts automatically.
3. The workflow identifies the PR author.
4. It checks whether `user2` is already requested.
5. If not, it requests a review from `user2`.
6. CODEOWNERS independently requests `user1`.
7. Branch protection requires the mandatory approval.

Important: This workflow is designed for ordinary PRs within the repository. Fork PRs, restricted Actions permissions, and repository policies may require additional configuration. Also, the workflow must be available in the base branch for the PR event to execute.

### 5.6 Alternative: Use GitHub Actions for multiple optional reviewers

If you want multiple optional reviewers, change:

```
const reviewers = [
  'user2',
  'user3',
  'user4'
];
```

Then use:

```
await github.rest.pulls.requestReviewers({
  owner,
  repo,
  pull_number,
  reviewers
});
```

Keep the PR-author and already-requested checks in place when adapting this version.

## 6. Configure Required and Optional Reviewers in GitHub Organization

For an Organization-owned repository, GitHub provides a more centralized approach using Rulesets and Required Reviewer Rules.

This is especially useful when managing multiple repositories and teams.

### 6.1 Create GitHub Organization teams

First, create teams in your GitHub Organization.

Example:

| Team                | Members            | Purpose               |
| ------------------- | ------------------ | --------------------- |
| Mandatory-Reviewers | user1              | Mandatory approval    |
| Optional-Reviewers  | user2              | Optional review       |
| DevOps-Team         | DevOps engineers   | Infrastructure review |
| DBA-Team            | Database engineers | SQL review            |

You can create teams from:

Organization → Teams → New team

Give the relevant teams Write access to the repositories where their reviews must count.

### 6.2 Create an Organization Ruleset

Navigate to:

1. Open your GitHub Organization.
2. Go to Settings.
3. Under Code, planning, and automation, select Repository → Rulesets.
4. Click New ruleset.
5. Select New branch ruleset.
6. Give the ruleset a name, for example:

```
Production Branch Review Policy
```

7. Select the target repositories.
8. Configure the target branches, such as `main`.
9. Set the enforcement status to Active after testing.

### 6.3 Configure branch protection rules in Rulesets

Enable:

- Require a pull request before merging.
- Require approvals.
- Require review from Code Owners, if using CODEOWNERS.
- Dismiss stale pull request approvals, if required.
- Require approval of the most recent reviewable push, if required.

### 6.4 Configure Required Reviewer Rules

This is the important feature for your use case.

Under the Ruleset's branch protection rules, configure Required reviewers.

You can define reviewer teams, file patterns, and the number of approvals required from each team.

Example configuration:

| Reviewer team       | File pattern   | Required approvals |
| ------------------- | -------------- | ------------------ |
| Mandatory-Reviewers | `*`            | 1                  |
| Optional-Reviewers  | `*`            | 0                  |
| DevOps-Team         | `terraform/**` | 1                  |
| DBA-Team            | `sql/**`       | 1                  |

Here is how it works:

- Mandatory-Reviewers requires one approval.
- Optional-Reviewers is added for visibility, but requires zero approvals.
- DevOps-Team must approve applicable Terraform changes.
- DBA-Team must approve applicable SQL changes.



Key feature

&#x20;GitHub's Required Reviewer Rules support zero approvals. This allows an organization to request a team's review without making its approval mandatory.



Important: Required Reviewer Rules are team-based, not individual-user-based. If you want to enforce an individual reviewer, create a dedicated team containing only that user.

Also, the team must have Write or higher repository permissions for its approval to count. Required Reviewer Rules are not available on user-owned repositories.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://docs.github.com\&sz=32)

GitHub Enterprise Cloud Docs+1



### 6.5 Organization-wide vs repository-level Rulesets

| Configuration                 | Organization-wide Ruleset | Repository Ruleset |
| ----------------------------- | ------------------------- | ------------------ |
| Scope                         | Multiple repositories     | One repository     |
| Centralized policy            | Yes                       | No                 |
| Target specific branches      | Yes                       | Yes                |
| Required reviewer teams       | Yes                       | Yes                |
| Individual reviewer selection | No                        | No                 |
| Team-based approval           | Yes                       | Yes                |

For organization-wide configuration, the reviewer teams must belong to the organization.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://docs.github.com\&sz=32)

GitHub Enterprise Cloud Docs+1



## 7. Comparison: GitHub Actions vs Organization Rulesets

| Feature                                  | GitHub Actions                       | Organization Rulesets           |
| ---------------------------------------- | ------------------------------------ | ------------------------------- |
| Automatically request optional reviewers | Yes                                  | Yes                             |
| Mandatory reviewer approval              | Through CODEOWNERS/branch protection | Through required reviewer rules |
| Optional reviewer approval               | Not mandatory by default             | Configure 0 approvals           |
| Individual users                         | Yes                                  | No, use single-member teams     |
| Teams                                    | Yes                                  | Yes                             |
| Centralized management                   | Workflow per repository              | Organization-wide               |
| File-specific review                     | Custom workflow logic                | Built-in patterns               |
| Recommended for                          | Personal repositories                | Organization repositories       |

## 8. Complete end-to-end PR review process

The following flow represents the recommended implementation.

\#chatgpt-mermaid-\_r_v3\_{font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";font-size:16px;fill:rgb(13, 13, 13);}@keyframes edge-animation-frame{from{stroke-dashoffset:0;}}@keyframes dash{to{stroke-dashoffset:0;}}#chatgpt-mermaid-\_r_v3\_ .edge-animation-slow{stroke-dasharray:9,5!important;stroke-dashoffset:900;animation:dash 50s linear infinite;stroke-linecap:round;}#chatgpt-mermaid-\_r_v3\_ .edge-animation-fast{stroke-dasharray:9,5!important;stroke-dashoffset:900;animation:dash 20s linear infinite;stroke-linecap:round;}#chatgpt-mermaid-\_r_v3\_ .error-icon{fill:rgb(243, 243, 243);}#chatgpt-mermaid-\_r_v3\_ .error-text{fill:rgb(13, 13, 13);stroke:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_v3\_ .edge-thickness-normal{stroke-width:1px;}#chatgpt-mermaid-\_r_v3\_ .edge-thickness-thick{stroke-width:3.5px;}#chatgpt-mermaid-\_r_v3\_ .edge-pattern-solid{stroke-dasharray:0;}#chatgpt-mermaid-\_r_v3\_ .edge-thickness-invisible{stroke-width:0;fill:none;}#chatgpt-mermaid-\_r_v3\_ .edge-pattern-dashed{stroke-dasharray:3;}#chatgpt-mermaid-\_r_v3\_ .edge-pattern-dotted{stroke-dasharray:2;}#chatgpt-mermaid-\_r_v3\_ .marker{fill:rgb(143, 143, 143);stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_v3\_ .marker.cross{stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_v3\_ svg{font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";font-size:16px;}#chatgpt-mermaid-\_r_v3\_ p{margin:0;}#chatgpt-mermaid-\_r_v3\_ .label{font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";color:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_v3\_ .cluster-label text{fill:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_v3\_ .cluster-label span{color:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_v3\_ .cluster-label span p{background-color:transparent;}#chatgpt-mermaid-\_r_v3\_ .label text,#chatgpt-mermaid-\_r_v3\_ span{fill:rgb(13, 13, 13);color:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_v3\_ .node rect,#chatgpt-mermaid-\_r_v3\_ .node circle,#chatgpt-mermaid-\_r_v3\_ .node ellipse,#chatgpt-mermaid-\_r_v3\_ .node polygon,#chatgpt-mermaid-\_r_v3\_ .node path{fill:rgb(222, 234, 251);stroke:rgb(83, 154, 248);stroke-width:1px;}#chatgpt-mermaid-\_r_v3\_ .rough-node .label text,#chatgpt-mermaid-\_r_v3\_ .node .label text,#chatgpt-mermaid-\_r_v3\_ .image-shape .label,#chatgpt-mermaid-\_r_v3\_ .icon-shape .label{text-anchor:middle;}#chatgpt-mermaid-\_r_v3\_ .node .katex path{fill:#000;stroke:#000;stroke-width:1px;}#chatgpt-mermaid-\_r_v3\_ .rough-node .label,#chatgpt-mermaid-\_r_v3\_ .node .label,#chatgpt-mermaid-\_r_v3\_ .image-shape .label,#chatgpt-mermaid-\_r_v3\_ .icon-shape .label{text-align:center;}#chatgpt-mermaid-\_r_v3\_ .node.clickable{cursor:pointer;}#chatgpt-mermaid-\_r_v3\_ .root .anchor path{fill:rgb(143, 143, 143)!important;stroke-width:0;stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_v3\_ .arrowheadPath{fill:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_v3\_ .edgePath .path{stroke:rgb(143, 143, 143);stroke-width:1px;}#chatgpt-mermaid-\_r_v3\_ .flowchart-link{stroke:rgb(143, 143, 143);fill:none;}#chatgpt-mermaid-\_r_v3\_ .edgeLabel{background-color:rgb(252, 252, 252);text-align:center;}#chatgpt-mermaid-\_r_v3\_ .edgeLabel p{background-color:rgb(252, 252, 252);}#chatgpt-mermaid-\_r_v3\_ .edgeLabel rect{opacity:0.5;background-color:rgb(252, 252, 252);fill:rgb(252, 252, 252);}#chatgpt-mermaid-\_r_v3\_ .labelBkg{background-color:rgba(252, 252, 252, 0.5);}#chatgpt-mermaid-\_r_v3\_ .cluster rect{fill:rgb(243, 243, 243);stroke:rgba(0, 0, 0, 0.1);stroke-width:1px;}#chatgpt-mermaid-\_r_v3\_ .cluster text{fill:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_v3\_ .cluster span{color:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_v3\_ div.mermaidTooltip{position:absolute;text-align:center;max-width:200px;padding:2px;font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";font-size:12px;background:rgb(243, 243, 243);border:1px solid rgba(0, 0, 0, 0.1);border-radius:2px;pointer-events:none;z-index:100;}#chatgpt-mermaid-\_r_v3\_ .flowchartTitleText{text-anchor:middle;font-size:18px;fill:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_v3\_ rect.text{fill:none;stroke-width:0;}#chatgpt-mermaid-\_r_v3\_ .icon-shape,#chatgpt-mermaid-\_r_v3\_ .image-shape{background-color:rgb(252, 252, 252);text-align:center;}#chatgpt-mermaid-\_r_v3\_ .icon-shape p,#chatgpt-mermaid-\_r_v3\_ .image-shape p{background-color:rgb(252, 252, 252);padding:2px;}#chatgpt-mermaid-\_r_v3\_ .icon-shape .label rect,#chatgpt-mermaid-\_r_v3\_ .image-shape .label rect{opacity:0.5;background-color:rgb(252, 252, 252);fill:rgb(252, 252, 252);}#chatgpt-mermaid-\_r_v3\_ .label-icon{display:inline-block;height:1em;overflow:visible;vertical-align:-0.125em;}#chatgpt-mermaid-\_r_v3\_ .node .label-icon path{fill:currentColor;stroke:revert;stroke-width:revert;}#chatgpt-mermaid-\_r_v3\_ .node .neo-node{stroke:rgb(83, 154, 248);}#chatgpt-mermaid-\_r_v3\_ [data-look="neo"].node rect,#chatgpt-mermaid-\_r_v3\_ [data-look="neo"].cluster rect,#chatgpt-mermaid-\_r_v3\_ [data-look="neo"].node polygon{stroke:url(#chatgpt-mermaid-\_r_v3\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_v3\_ [data-look="neo"].swimlane.cluster rect{filter:none;}#chatgpt-mermaid-\_r_v3\_ [data-look="neo"].node path{stroke:url(#chatgpt-mermaid-\_r_v3\_-gradient);stroke-width:1px;}#chatgpt-mermaid-\_r_v3\_ [data-look="neo"].node .outer-path{filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_v3\_ [data-look="neo"].node .neo-line path{stroke:rgb(83, 154, 248);filter:none;}#chatgpt-mermaid-\_r_v3\_ [data-look="neo"].node circle{stroke:url(#chatgpt-mermaid-\_r_v3\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_v3\_ [data-look="neo"].node circle .state-start{fill:#000000;}#chatgpt-mermaid-\_r_v3\_ [data-look="neo"].icon-shape .icon{fill:url(#chatgpt-mermaid-\_r_v3\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_v3\_ [data-look="neo"].icon-shape .icon-neo path{stroke:url(#chatgpt-mermaid-\_r_v3\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_v3\_ .node text{font-size:14px;font-weight:600;letter-spacing:normal;fill:rgb(0, 79, 153);}#chatgpt-mermaid-\_r_v3\_ .edgeLabels text{font-size:13px;font-weight:600;letter-spacing:-0.08px;fill:rgb(0, 79, 153);}#chatgpt-mermaid-\_r_v3\_ .node tspan[font-weight="normal"],#chatgpt-mermaid-\_r_v3\_ .edgeLabels tspan[font-weight="normal"]{font-weight:600;}#chatgpt-mermaid-\_r_v3\_ .edgeLabel .label rect{opacity:1;rx:13px;ry:13px;fill:rgb(245, 250, 255);stroke:rgb(206, 219, 229);stroke-width:1px;}#chatgpt-mermaid-\_r_v3\_ .node rect,#chatgpt-mermaid-\_r_v3\_ .node circle,#chatgpt-mermaid-\_r_v3\_ .node ellipse,#chatgpt-mermaid-\_r_v3\_ .node polygon,#chatgpt-mermaid-\_r_v3\_ .node path{fill:rgb(229, 243, 255);stroke:rgba(0, 0, 0, 0.1);stroke-width:1px;}#chatgpt-mermaid-\_r_v3\_ .node rect{rx:16px;ry:16px;}#chatgpt-mermaid-\_r_v3\_ .mermaid-decision .label-container{fill:rgb(245, 250, 255);stroke:rgb(206, 219, 229);stroke-dasharray:2,2;}#chatgpt-mermaid-\_r_v3\_ .edgePaths .flowchart-link{stroke:rgb(143, 143, 143);stroke-width:1px;stroke-linecap:round;stroke-linejoin:round;}#chatgpt-mermaid-\_r_v3\_ .marker{fill:rgb(143, 143, 143);stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_v3\_ :root{--mermaid-font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";}Developer creates PullRequestGitHub checks CODEOWNERSRequest mandatory revieweruser1GitHub Actions requestsoptional reviewer user2Wait for mandatory approvalOptional review can happenuser1 approved?Merge blockedOther required checkspassed?Merge allowedNoYesNoYes

For an Organization repository using Rulesets, the required and optional teams can instead be requested directly through Required Reviewer Rules.

## 9. Recommended production configuration

For a production-grade DevOps repository, I recommend the following:

| Area                    | Recommended configuration |
| ----------------------- | ------------------------- |
| Default branch          | `main`                    |
| CODEOWNERS              | Mandatory reviewer        |
| Required approvals      | Minimum 1                 |
| Optional reviewers      | Automatically assigned    |
| Stale approvals         | Dismiss on new commits    |
| Direct push             | Restricted                |
| Force push              | Disabled                  |
| Branch deletion         | Disabled                  |
| CI/CD status checks     | Required                  |
| Conversation resolution | Required                  |
| Admin bypass            | Restricted                |

### Example final CODEOWNERS

```
# Mandatory repository reviewer
* @user1

# Infrastructure ownership
/terraform/ @devops-team

# Kubernetes ownership
/kubernetes/ @platform-team

# Database ownership
/sql/ @dba-team
```

Remember that enabling Require review from Code Owners makes the applicable owners mandatory. If the same file has multiple owners, approval from any one of those owners satisfies the code-owner requirement.

For more granular organization policies, use Required Reviewer Rules instead of treating all CODEOWNERS as mandatory.

## 10. Troubleshooting

| Problem                              | Possible reason                                | Resolution                                   |
| ------------------------------------ | ---------------------------------------------- | -------------------------------------------- |
| CODEOWNERS reviewer not assigned     | File missing from base branch                  | Commit CODEOWNERS to target branch           |
| CODEOWNERS file invalid              | Incorrect syntax                               | Validate username, pattern and team          |
| Optional reviewer not assigned       | Workflow not triggered                         | Check PR event configuration                 |
| Actions cannot request reviewer      | Insufficient token permissions                 | Enable `pull-requests: write`                |
| Reviewer cannot approve              | No repository access                           | Grant appropriate repository permissions     |
| PR merges without mandatory approval | Protection rule not active or bypass permitted | Review branch protection and bypass settings |
| Team approval does not count         | Team lacks Write permission                    | Grant team repository access                 |
| Reviewer not assigned for draft PR   | CODEOWNERS notifications are deferred          | Mark PR ready for review                     |

Additional documentation:

- [CODEOWNERS](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-code-owners?utm_source=chatgpt.com)
- [Branch Protection](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches?utm_source=chatgpt.com)
- [Available Ruleset Rules](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/available-rules-for-rulesets?utm_source=chatgpt.com)
- [Organization Rulesets](https://docs.github.com/en/organizations/managing-organization-settings/creating-rulesets-for-repositories-in-your-organization?utm_source=chatgpt.com)
- [GitHub Actions Workflow Syntax](https://docs.github.com/en/actions/writing-workflows/workflow-syntax-for-github-actions?utm_source=chatgpt.com)

## 11. Final recommendation

Choose the implementation based on your repository ownership:

Personal GitHub repository

Recommended approach

- CODEOWNERS for mandatory reviewer.
- Branch Protection for required approval.
- GitHub Actions for automatically requesting optional reviewers.

  GitHub Organization repository

Recommended approach

- Organization teams for reviewer management.
- Rulesets for centralized branch protection.
- Required Reviewer Rules with 1 approval for mandatory teams.
- Required Reviewer Rules with 0 approvals for optional teams.
- CODEOWNERS for file ownership and automatic review requests.

Important: Organization Rulesets and Required Reviewer Rules can depend on your GitHub plan and feature availability. Verify availability in your organization's settings before adopting them.

The key distinction is that the personal-account approach uses GitHub Actions to automate optional reviewer assignment, whereas the Organization approach can manage required and optional team reviews natively through Rulesets.
