# Figma-GitHub-Jira Integration Workflow

This project provides an automated workflow that bridges Figma, GitHub, and Jira to streamline bug reporting and tracking. When a bug issue is created on GitHub (either manually or via Figma plugins), it automatically creates a corresponding Jira ticket with extracted details using AI.

## Overview

The integration works as follows:

1. **Figma → GitHub**: Use Figma plugins to create GitHub issues directly from your designs
2. **GitHub → Jira**: Automatically create Jira tickets from GitHub bug issues using AI-powered extraction
3. **AI Processing**: Extracts structured information (priority, Figma links, attachments, description) from issue body

## Figma Plugins

To create GitHub issues from Figma, install one of these plugins:

- **GitHub Issue Sync**: [https://www.figma.com/community/widget/1101944648482192724](https://www.figma.com/community/widget/1101944648482192724)
- **Issue to GitHub**: [https://www.figma.com/community/widget/1306216732737885211](https://www.figma.com/community/widget/1306216732737885211)

These plugins allow you to:
- Select elements in Figma and create GitHub issues with attached screenshots
- Automatically include Figma frame/selection links
- Tag issues with relevant labels

## Setup

### Prerequisites

- A GitHub repository with Actions enabled
- A Jira Cloud instance with API access
- OpenAI API access (via GitHub Models)

### Installation

1. Copy the `.github/workflows/issue-to-jira.yml` file to your repository
2. Copy the `prompts/issue-extract.prompt.yml` file to your repository
3. Configure the required secrets and variables (see below)

## GitHub Action Configuration

### Required Secrets

Configure these secrets in your GitHub repository settings (`Settings → Secrets and variables → Actions → Secrets`):

| Secret Name | Description |
|-------------|-------------|
| `JIRA_API_TOKEN` | Your Jira API token. Generate at [https://id.atlassian.com/manage-profile/security/api-tokens](https://id.atlassian.com/manage-profile/security/api-tokens) |

### Required Variables

Configure these variables in your GitHub repository settings (`Settings → Secrets and variables → Actions → Variables`):

| Variable Name | Description | Example |
|---------------|-------------|---------|
| `JIRA_BASE_URL` | Your Jira Cloud instance URL | `https://yourcompany.atlassian.net` |
| `JIRA_EMAIL` | Email address associated with your Jira account | `you@example.com` |
| `JIRA_PROJECT_KEY` | The project key where tickets will be created | `PROJ`, `BUG`, `DEV` |

## Workflow Details

The workflow file `.github/workflows/issue-to-jira.yml` performs the following steps:

### Trigger
```yaml
on:
  issues:
    types: [opened]
```
The workflow triggers whenever a new issue is opened in the repository.

### Job Configuration
```yaml
jobs:
  sync-bug-to-jira:
    environment: development
    if: contains(github.event.issue.labels.*.name, 'bug')
```
Only processes issues labeled with `bug`.

### Step-by-Step Breakdown

#### 1. Checkout Repository
```yaml
- name: Checkout repository
  uses: actions/checkout@v4
```
Checks out the repository to access the prompt file.

#### 2. Prepare Issue Body
```yaml
- name: Prepare single-line body
  id: prepare
  run: |
    BODY="${{ github.event.issue.body }}"
    SINGLE_LINE=$(echo "$BODY" | tr '\n' ' ' | sed 's/  */ /g')
    echo "body=$SINGLE_LINE" >> $GITHUB_OUTPUT
```
Converts the multi-line issue body into a single line by replacing newlines with spaces. This ensures proper handling by the AI inference action.

#### 3. Extract Issue Details with AI
```yaml
- name: Extract issue details
  id: inference
  uses: actions/ai-inference@v1
  with:
    prompt-file: './prompts/issue-extract.prompt.yml'
    input: |
      body: ${{ toJSON(steps.prepare.outputs.body) }}
```
Uses GitHub's AI inference action with GPT-4o to extract structured data from the issue body. See the [Prompt Configuration](#prompt-configuration) section for details on the extraction schema.

#### 4. Login to Jira
```yaml
- name: Login
  uses: atlassian/gajira-login@v3
  env:
    JIRA_BASE_URL: ${{ vars.JIRA_BASE_URL }}
    JIRA_USER_EMAIL: ${{ vars.JIRA_EMAIL }}
    JIRA_API_TOKEN: ${{ secrets.JIRA_API_TOKEN }}
```
Authenticates with Jira using the configured credentials.

#### 5. Create Jira Ticket
```yaml
- name: Create Jira ticket
  id: create
  uses: atlassian/gajira-create@v3
  with:
    project: ${{ vars.JIRA_PROJECT_KEY }}
    issuetype: Bug
    summary: ${{ github.event.issue.title }}
    description: |
      h2. Priority
      *h3. ${{ fromJSON(steps.inference.outputs.response).priority }}*
      
      h2. GitHub issue
      [issue #${{ github.event.issue.number }}|${{ github.event.issue.url }}]
      
      h2. Summary
      ${{ fromJSON(steps.inference.outputs.response).content }}

      h2. Figma Links
      ${{ join(fromJSON(steps.inference.outputs.response).figma_urls, ' , ') }}
      
      h2. Attachments
      !${{ join(fromJSON(steps.inference.outputs.response).attachment_urls, '!!') }}!
```
Creates a Jira ticket with:
- **Project**: From `JIRA_PROJECT_KEY` variable
- **Issue Type**: Bug
- **Summary**: The GitHub issue title
- **Description**: Formatted with extracted priority, summary, Figma links, and attachment URLs

#### 6. Store Jira URL
```yaml
- name: Store Jira URL
  if: steps.create.outputs.issue != ''
  run: |
    echo "JIRA_KEY=${{ steps.create.outputs.issue }}" >> $GITHUB_ENV
    echo "JIRA_URL=${{ vars.JIRA_BASE_URL }}/browse/${{ steps.create.outputs.issue }}" >> $GITHUB_ENV
```
Stores the created Jira ticket key and URL for later use.

#### 7. Comment on GitHub Issue
```yaml
- name: Comment Jira link on GitHub issue
  if: env.JIRA_URL != ''
  uses: actions/github-script@v7
  with:
    script: |
      const body = `**Jira ticket** [${{ env.JIRA_KEY }}](${{ env.JIRA_URL }}) created via GitHub Action.`;
      
      await github.rest.issues.createComment({
        issue_number: context.issue.number,
        owner: context.repo.owner,
        repo: context.repo.repo,
        body
      });
```
Adds a comment to the original GitHub issue with a link to the newly created Jira ticket.

## Prompt Configuration

The `prompts/issue-extract.prompt.yml` file configures the AI extraction:

### Model Settings
- **Model**: `openai/gpt-4o`
- **Temperature**: `0.5` (balanced creativity and consistency)

### Extraction Schema

The AI extracts the following fields from issue body:

| Field | Type | Description |
|-------|------|-------------|
| `content` | string | Complete issue description including reproduction steps |
| `priority` | string | One of: `Critical`, `Non-Critical`, `Cosmetic`, `Nice to have` |
| `figma_urls` | array | List of Figma URLs found in the issue |
| `attachment_urls` | array | List of non-Figma attachment URLs |

### Example Output
```json
{
  "content": "Login button doesn't respond on mobile view. Steps: 1. Open login page 2. Resize to mobile 3. Click button",
  "priority": "Critical",
  "figma_urls": ["https://www.figma.com/file/abc123/Login-Page"],
  "attachment_urls": ["https://user-images.githubusercontent.com/.../screenshot.png"]
}
```

## Usage

### Creating Issues via Figma

1. Select the element(s) in Figma you want to report
2. Open the Figma plugin
3. Fill in issue details
4. Submit to create a GitHub issue (automatically labeled as `bug`)

### Creating Issues Manually

1. Create a new GitHub issue
2. Add the `bug` label
3. Include Figma links in the body if applicable
4. Attach screenshots if needed
5. Submit - the workflow will automatically create a Jira ticket

## Permissions

The workflow requires these permissions:

```yaml
permissions:
  issues: write        # To comment on GitHub issues
  contents: read       # To checkout repository
  models: read         # To use GitHub AI inference
```

## Troubleshooting

### Issue not creating Jira ticket
- Ensure the issue has the `bug` label
- Check that all required secrets and variables are configured
- Verify the Jira API token hasn't expired

### AI extraction not working
- Ensure the prompt file exists at `prompts/issue-extract.prompt.yml`
- Check GitHub Models access is enabled for your repository

### Jira authentication failed
- Verify `JIRA_BASE_URL` matches your Jira Cloud URL exactly
- Regenerate your Jira API token if needed
- Ensure the email matches your Jira account

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

## Support

For issues or questions:
1. Check the [Troubleshooting](#troubleshooting) section
2. Open an issue in this repository
3. Review the [GitHub Actions documentation](https://docs.github.com/en/actions)
4. Check the [Jira API documentation](https://developer.atlassian.com/cloud/jira/platform/rest/v3/)
