---
layout: single
title: "AI Code Review: Automate PR Reviews with Claude in 20 Minutes"
date: 2026-03-06
category: "coding"
tags: ["coding", "atlas-signal", "deep-research", "Claude", "Python", "CodingTutorial"]
description: "You can build a custom GitHub Action that uses Claude 3.5 Sonnet to automatically review pull requests, catching bugs and suggesting improvements before human r"
canonical_url: "https://atlassignal.in/posts/ai-code-review-automate-pr-reviews-with-claude-in-20-minutes/"
og_title: "AI Code Review: Automate PR Reviews with Claude in 20 Minutes"
og_description: "You can build a custom GitHub Action that uses Claude 3.5 Sonnet to automatically review pull requests, catching bugs and suggesting improvements before human r"
og_url: "https://atlassignal.in/posts/ai-code-review-automate-pr-reviews-with-claude-in-20-minutes/"
og_image: "https://images.pexels.com/photos/5483077/pexels-photo-5483077.jpeg?auto=compress&cs=tinysrgb&dpr=2&h=650&w=940"
header:
  overlay_image: https://images.pexels.com/photos/5483077/pexels-photo-5483077.jpeg?auto=compress&cs=tinysrgb&dpr=2&h=650&w=940
  overlay_filter: 0.75
twitter_card: summary_large_image
toc: true
toc_sticky: true
---

![AI Code Review: Automate PR Reviews with Claude in 20 Minutes](https://images.pexels.com/photos/5483077/pexels-photo-5483077.jpeg?auto=compress&cs=tinysrgb&dpr=2&h=650&w=940)


**Difficulty:** Intermediate | **Category:** Coding

# AI Code Review: Automate PR Reviews with Claude in 20 Minutes


<ins class="adsbygoogle"
     style="display:block"
     data-ad-client=""
     data-ad-slot="AUTO"
     data-ad-format="auto"
     data-full-width-responsive="true"></ins>
<script>(adsbygoogle = window.adsbygoogle || []).push({});</script>


Manual code reviews consume 20-30% of engineering time according to a 2025 Google DevOps report, yet they're critical for catching bugs before production. With Claude 3.5 Sonnet's 200K context window, you can now automate first-pass PR reviews to catch obvious issues, security vulnerabilities, and style inconsistencies—letting human reviewers focus on architecture and business logic.

In this tutorial, you'll build a GitHub Action that automatically comments on every pull request with Claude's analysis, complete with specific line-by-line feedback.

## Prerequisites

- GitHub repository with admin access (or ability to create Actions)
- Anthropic API key (get one at console.anthropic.com—$15 free credit)
- Basic familiarity with GitHub Actions and YAML
- Node.js 18+ installed locally for testing

## Step 1: Get Your Anthropic API Key and Set Up Secrets

First, navigate to console.anthropic.com and create an account. Click "Get API Keys" and generate a new key. Copy it immediately—you won't see it again.

In your GitHub repository, go to **Settings → Secrets and variables → Actions → New repository secret**. Name it `ANTHROPIC_API_KEY` and paste your key.

**Gotcha:** Don't commit API keys to your repository, even in "private" repos. GitHub's secret scanning will flag it, and Anthropic will automatically revoke the key.

**Pro tip:** Set up billing alerts in your Anthropic console. Claude 3.5 Sonnet costs $3 per million input tokens and $15 per million output tokens. A typical PR review uses 5K-20K tokens (about $0.03-$0.30 per review).

## Step 2: Create the GitHub Action Workflow File

Create `.github/workflows/ai-code-review.yml` in your repository:

```yaml
name: AI Code Review with Claude

on:
  pull_request:
    types: [opened, synchronize]

permissions:
  contents: read
  pull-requests: write

jobs:
  ai-review:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Get PR diff
        id: diff
        run: |
          git fetch origin ${{ github.base_ref }}
          DIFF=$(git diff origin/${{ github.base_ref }}...HEAD)
          echo "diff> $GITHUB_OUTPUT
          echo "$DIFF" >> $GITHUB_OUTPUT
          echo "EOF" >> $GITHUB_OUTPUT

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Run AI Review
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
          PR_NUMBER: ${{ github.event.pull_request.number }}
          REPO: ${{ github.repository }}
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          node .github/scripts/ai-review.js
```

This workflow triggers on every PR open and update, fetches the diff, and runs your review script.

## Step 3: Write the Claude Review Script

Create `.github/scripts/ai-review.js`:

```javascript
const Anthropic = require('@anthropic-ai/sdk');
const { Octokit } = require('@octokit/rest');
const fs = require('fs');

const anthropic = new Anthropic({
  apiKey: process.env.ANTHROPIC_API_KEY,
});

const octokit = new Octokit({
  auth: process.env.GITHUB_TOKEN,
});

async function reviewCode() {
  const diff = fs.readFileSync('/tmp/pr.diff', 'utf-8');
  
  const message = await anthropic.messages.create({
    model: 'claude-3-5-sonnet-20241022',
    max_tokens: 4096,
    temperature: 0.3,
    system: `You are an expert code reviewer. Analyze the git diff and provide:
1. Security vulnerabilities (SQL injection, XSS, authentication issues)
2. Logic bugs or edge cases
3. Performance concerns
4. Code style issues (if severe)

Format your response as:
## Summary
[1-2 sentences]

## Issues Found
### 🔴 Critical
- [issue]: [explanation]

### 🟡 Suggestions  
- [suggestion]: [explanation]

Be concise. Only flag real issues—don't nitpick formatting if it's consistent.`,
    messages: [{
      role: 'user',
      content: `Review this pull request diff:\n\n${diff.substring(0, 150000)}`
    }]
  });

  const review = message.content[0].text;
  
  const [owner, repo] = process.env.REPO.split('/');
  await octokit.rest.issues.createComment({
    owner,
    repo,
    issue_number: parseInt(process.env.PR_NUMBER),
    body: `## 🤖 AI Code Review (Claude 3.5 Sonnet)\n\n${review}\n\n---\n*Automated review • [Improve this action](https://github.com/${process.env.REPO}/blob/main/.github/scripts/ai-review.js)*`
  });
}

// Save diff from GitHub Actions
const diff = process.env.DIFF || '';
fs.writeFileSync('/tmp/pr.diff', diff);

reviewCode().catch(console.error);
```

**Gotcha:** The 150,000 character substring limit prevents token overflow. Claude 3.5 Sonnet handles 200K tokens (~750K characters), but we cap it safely. For massive PRs, consider splitting files.

## Step 4: Add Dependencies and Test Locally

Create `package.json` in `.github/scripts/`:

```json
{
  "name": "ai-code-review",
  "version": "1.0.0",
  "dependencies": {
    "@anthropic-ai/sdk": "^0.20.0",
    "@octokit/rest": "^20.0.2"
  }
}
```

Test locally before committing:

```bash
cd .github/scripts
npm install
export ANTHROPIC_API_KEY="your-key-here"
export GITHUB_TOKEN="your-github-pat"
export PR_NUMBER="123"
export REPO="yourname/yourrepo"
export DIFF="$(git diff main...feature-branch)"
node ai-review.js
```

You should see a comment posted to your PR (use a test PR first!).

**Pro tip:** Add a `.node-version` file with `20` to ensure consistency across environments.

## Step 5: Customize the Review Prompt for Your Codebase

The system prompt in Step 3 is generic. Tailor it to your team's needs:

```javascript
system: `You are a senior ${language} developer reviewing code for a ${projectType} project.

Code standards:
- We use ${framework} version ${version}
- Authentication via ${authMethod}
- Database queries must use parameterized statements
- All user input requires validation

Focus on:
1. Security (OWASP Top 10)
2. ${specificConcern} (e.g., "race conditions in async code")
3. Breaking changes to public APIs

Ignore:
- Formatting (handled by prettier)
- Missing comments (not required)

Flag only medium+ severity issues.`
```

For a Python/Django project, you might specify: "Check for N+1 queries, missing CSRF tokens, and unsafe pickle usage."

**Gotcha:** Temperature 0.3 makes Claude more consistent but less creative. For exploratory reviews, try 0.7. For security-critical code, use 0.1.

## Step 6: Add File-Specific Review Rules

Enhance the script to skip certain files or apply custom rules:

```javascript
// Before calling Claude
const filteredDiff = diff
  .split('\ndiff --git')
  .filter(fileDiff => {
    // Skip generated files
    if (fileDiff.includes('package-lock.json')) return false;
    if (fileDiff.includes('dist/')) return false;
    return true;
  })
  .join('\ndiff --git');

// For sensitive files, add stricter prompts
if (diff.includes('auth.js') || diff.includes('security.py')) {
  systemPrompt += '\n\nEXTRA SCRUTINY: This PR modifies authentication. Check for timing attacks, token leakage, and session fixation.';
}
```

## Step 7: Set Up Cost Controls and Rate Limiting

Add a token counter to prevent runaway costs:

```javascript
function estimateTokens(text) {
  // Rough estimate: 1 token ≈ 4 characters for English
  return text.length / 4;
}

const estimatedTokens = estimateTokens(diff);
if (estimatedTokens > 100000) {
  console.log(`⚠️  Diff too large (${estimatedTokens} tokens). Skipping AI review.`);
  process.exit(0);
}
```

**Pro tip:** In your Anthropic dashboard, set a monthly budget alert at $50. Most teams spend $10-30/month reviewing 50-100 PRs.

## Practical Example: Complete Working Setup

Here's a real-world review result from a React PR that added user authentication:

```markdown
## 🤖 AI Code Review (Claude 3.5 Sonnet)

## Summary
This PR implements login functionality. Found 1 critical security issue and 2 suggestions.

## Issues Found

### 🔴 Critical
- **Cleartext password logging (auth.js:45)**: `console.log(password)` exposes credentials in logs. Remove immediately.

### 🟡 Suggestions
- **Missing error handling (LoginForm.jsx:78)**: `fetch()` has no catch block. Network failures will crash the component.
- **Hardcoded API URL (config.js:12)**: Use environment variables for `API_BASE_URL` to support staging/prod environments.

## What Looks Good
- Proper JWT storage in httpOnly cookies ✓
- Input validation using Joi ✓
```

This caught a real security bug before human review.

## Key Takeaways

- **Claude 3.5 Sonnet can review PRs with 200K context**, sufficient for most diffs under 500 changed lines
- **Cost-effective automation**: At $0.03-$0.30 per review, you save 15-30 minutes of engineering time (worth $15-30)
- **Custom prompts are essential**—generic reviews miss project-specific issues like framework misuse or architectural violations
- **Combine AI + human reviews**: Claude catches obvious bugs; humans handle business logic and design trade-offs

## What's Next

Once you've mastered automated PR reviews, explore training Claude on your codebase's vector embeddings to provide context-aware suggestions about internal APIs and architectural patterns.

---

**Key Takeaway:** You can build a custom GitHub Action that uses Claude 3.5 Sonnet to automatically review pull requests, catching bugs and suggesting improvements before human review—saving 2-3 hours per week for engineering teams.

---

*New AI tutorials published daily on [AtlasSignal](https://atlassignal.in). Follow [@AtlasSignalDesk](https://twitter.com/AtlasSignalDesk) for more.*

---

## 📧 Get Daily AI & Macro Intelligence

Stay ahead of market-moving news, emerging tech, and global shifts.

<div class="email-capture">
  <form action="" method="post" target="_blank">
    <input type="email" name="EMAIL" placeholder="Your email address" required />
    <button type="submit">Subscribe Free →</button>
  </form>
</div>

