# GitHub Security Review Bot

A GitHub Actions-based security review bot that automatically analyzes pull requests for security vulnerabilities using AI-powered analysis. The bot analyzes code changes and provides detailed security feedback with OWASP and CWE references.

## Features

- 🔍 **Automated Security Review**

  - Reviews pull requests automatically
  - Analyzes code changes for security vulnerabilities
  - Provides detailed feedback with OWASP and CWE references
  - Fully configurable security checks and thresholds

- 🤖 **AI-Powered Analysis**

  - Uses GPT-4 for intelligent code analysis
  - Detects complex security vulnerabilities
  - Provides context-aware suggestions
  - Configurable confidence thresholds

- 📊 **Configurable Security Checks**

  - Define your own security checks
  - Configurable severity levels and thresholds
  - Extensible check framework
  - Each check can be enabled/disabled independently
  - Custom OWASP and CWE mappings
  - Custom detection patterns and validation rules

- 🎯 **False Positive Handling**

  - Mark vulnerabilities as false positives via comments
  - Maintains history of false positives
  - Requires maintainer approval
  - Configurable expiration period

- 📝 **Detailed Reporting**
  - Severity levels (critical, high, medium, low)
  - Confidence scores
  - OWASP and CWE references
  - Suggested fixes
  - Location in code

## Configuration Files

### 1. GitHub Actions Workflow (.github/workflows/security-bot.yml)

This file defines the GitHub Actions workflow that runs the security review bot. Here's what it does:

```yaml
# Key components:
- Triggers on pull request events (opened, synchronize, reopened)
- Triggers on issue comments (for handling false positives)
- Sets up Node.js environment
- Installs dependencies
- Runs security analysis
- Reports results using GitHub Checks API
- Uploads logs as artifacts
```

### 2. Security Bot Configuration (security-bot/config/securitybot-config.yml)

This file controls the behavior of the security review bot. You can customize it based on your security requirements:

```yaml
# AI Model Configuration
ai_model:
  provider: "openai"
  model: "gpt-4"
  temperature: 0.3
  max_tokens: 2000

# Security Checks Configuration
security_checks:
  # Example security check
  sql_injection:
    enabled: true
    severity: critical
    description: "Detects potential SQL injection vulnerabilities"
    owasp: "A03:2021"
    cwe: "CWE-89"
    confidence_threshold: 0.8
    patterns:
      - "pattern1"
      - "pattern2"
    validation:
      - "rule1"
      - "rule2"

# Severity Levels
severity_levels:
  critical:
    enabled: true
    threshold: 0.9
    color: "#dc3545" # Bootstrap danger red
    description: "Critical security issues that must be addressed immediately"
  high:
    enabled: true
    threshold: 0.8
    color: "#fd7e14" # Bootstrap warning orange
    description: "High severity security issues"
  medium:
    enabled: true
    threshold: 0.7
    color: "#ffc107" # Bootstrap warning yellow
    description: "Medium severity security issues"
  low:
    enabled: true
    threshold: 0.6
    color: "#20c997" # Bootstrap success green
    description: "Low severity security issues"

# Performance Settings
max_lines: 1000
max_retries: 3
api_timeout: 30000

# Logging Configuration
logging:
  level: debug
  file: security-bot.log
  max_size: 10485760 # 10MB
  max_files: 5

# False Positive Management
false_positive:
  enabled: true
  command: "@[Ss]ecurity[Bb]ot false-positive"
  storage: "comments"
  expiration: 30
  require_approval: true
  track_history: true
  include_reason: true
```

## Installation and Setup

1. **Create Project Structure**
   First, create the following directory structure in your project:

   ```
   your-project/              # Your existing project root
   ├── .github/
   │   └── workflows/        # GitHub Actions workflows
   └── security-bot/         # Bot code and configuration
       ├── config/           # Bot configuration
       └── src/             # Source code
   ```

   ```bash
   # Create directories
   mkdir -p .github/workflows
   mkdir -p security-bot/config
   mkdir -p security-bot/src
   ```

2. **Set Up GitHub Actions Workflow**
   Create `.github/workflows/security-bot.yml` with the following content:

   ```yaml
   # GitHub Actions workflow for automated security review of pull requests
   # This workflow runs security analysis on code changes and provides feedback through PR comments

   name: Security Review Bot

   # Trigger conditions for the workflow
   on:
     # Run when pull requests are created or updated
     pull_request:
       types: [opened, synchronize, reopened]
     # Run when comments are added (for handling false positive marks)
     issue_comment:
       types: [created]

   # Add top-level permissions to ensure they are properly set
   permissions:
     contents: read
     pull-requests: write
     issues: write
     statuses: write
     checks: write

   jobs:
     # Main security review job
     security-review:
       name: Security Review
       runs-on: ubuntu-latest

       # Environment variables available to all steps
       env:
         GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }} # For GitHub API access
         OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }} # For AI-powered analysis
         NODE_VERSION: "18" # Node.js version to use

       steps:
         # Step 1: Check out the repository code
         - name: Checkout repository
           uses: actions/checkout@v4
           with:
             fetch-depth: 0 # Full history needed for accurate diff analysis
             token: ${{ secrets.GITHUB_TOKEN }}

         # Step 2: Set up Node.js environment
         - name: Setup Node.js
           uses: actions/setup-node@v4
           with:
             node-version: ${{ env.NODE_VERSION }}

         # Step 3: Install project dependencies
         - name: Install dependencies
           working-directory: security-bot
           run: |
             if [ -f "package-lock.json" ]; then
               npm ci
             else
               npm install
             fi

         # Step 4: Run the security analysis
         - name: Run security review
           id: security-review
           working-directory: security-bot
           run: node src/index.js
           env:
             # Pass through required GitHub context
             GITHUB_REPOSITORY: ${{ github.repository }}
             GITHUB_EVENT_PATH: ${{ github.event_path }}
             GITHUB_EVENT_NAME: ${{ github.event_name }}
             GITHUB_ACTIONS: true
             PR_NUMBER: ${{ github.event.pull_request.number }}
             CONFIG_PATH: config/securitybot-config.yml
           continue-on-error: false # Fail the workflow if security issues are found

         # Step 5: Report analysis results using checks API instead of statuses
         - name: Report status
           if: always() # Run this step regardless of previous step outcomes
           uses: actions/github-script@v7
           with:
             script: |
               const conclusion = process.env.GITHUB_STEP_SUMMARY.includes('Security vulnerabilities detected') ? 'failure' : 'success';
               const { data: checkRun } = await github.rest.checks.create({
                 owner: context.repo.owner,
                 repo: context.repo.repo,
                 name: 'Security Review',
                 head_sha: context.sha,
                 status: 'completed',
                 conclusion: conclusion,
                 output: {
                   title: 'Security Review Results',
                   summary: conclusion === 'success'
                     ? '✅ No security vulnerabilities found.'
                     : '❌ Security vulnerabilities detected. Please review the findings.'
                 }
               });

         # Step 6: Save log files as artifacts
         - name: Upload logs
           if: always() # Always upload logs for debugging
           uses: actions/upload-artifact@v4
           with:
             name: security-review-logs
             path: security-bot/*.log
             retention-days: 7 # Keep logs for 7 days

         # Step 7: Clean up temporary files
         - name: Cleanup
           if: always() # Always perform cleanup
           run: |
             # Remove log files
             rm -f security-bot/*.log
             rm -f security-bot/error.log
             # Remove node_modules to reduce storage usage
             rm -rf security-bot/node_modules
   ```

3. **Set Up Bot Configuration**
   Create `security-bot/config/securitybot-config.yml` with your security requirements. You can use the example configuration as a starting point and customize it based on your needs.

4. **Set Up Source Files**
   Create the following files in the `security-bot/src` directory:

   - `index.js` - Main bot implementation
   - `config.js` - Configuration loader
   - `logger.js` - Logging utility

5. **Set Up Package Dependencies**
   Create `security-bot/package.json`:

   ```json
   {
     "name": "github-security-review-bot",
     "version": "1.0.0",
     "dependencies": {
       "@octokit/rest": "^20.0.0",
       "openai": "^4.0.0",
       "js-yaml": "^4.1.0",
       "winston": "^3.11.0",
       "dotenv": "^16.3.1"
     }
   }
   ```

6. **Configure GitHub Secrets**
   Add the following secrets to your GitHub repository:
   - `GITHUB_TOKEN`: GitHub API token (automatically provided)
   - `OPENAI_API_KEY`: Your OpenAI API key

## Usage

The security review bot runs automatically when:

1. A new pull request is created
2. A pull request is updated
3. A comment is added to a pull request (for handling false positives)

### Example Workflow

1. **Configure Your Repository**

   - Add the security bot configuration to your repository
   - Set up the GitHub Actions workflow
   - Configure the required secrets

2. **Create a Pull Request**

   - Make changes to your code
   - Create a new pull request
   - The security review bot will automatically analyze your changes

3. **Review Security Findings**

   - The bot will comment on your PR with security findings
   - Each finding includes:
     - Vulnerability type
     - Severity level
     - Location in code
     - Suggested fixes
     - OWASP and CWE references

4. **Handle False Positives**
   If you find a false positive, comment on the PR with:

   ```
   @securitybot false-positive "Vulnerability Type" (file.js:line)
   ```

   Example:

   ```
   @securitybot false-positive SQL Injection (server.js:15)
   ```

   Requirements:

   - Use the exact vulnerability type as shown in the report
   - Include the parentheses around the location
   - Use the colon between file and line number
   - Keep the space between type and location
   - Must be a repository maintainer if `require_approval` is enabled

## Security Checks

You can define your own security checks in the configuration file. Here's how to set up a security check:

1. **Define a New Check**

   ```yaml
   security_checks:
     custom_check:
       enabled: true
       severity: high
       description: "Your custom check description"
       owasp: "A01:2021"
       cwe: "CWE-123"
       confidence_threshold: 0.7
   ```

2. **Configure Severity Levels**

   ```yaml
   severity_levels:
     custom_severity:
       enabled: true
       threshold: 0.85
       color: "#fd7e14" # Bootstrap warning orange
       description: "Custom severity description"
   ```

3. **Add Custom Validation Rules**
   ```yaml
   security_checks:
     custom_check:
       validation:
         - "rule1"
         - "rule2"
   ```

Each security check can be:

- Enabled or disabled independently
- Configured with custom severity levels
- Set with custom confidence thresholds
- Given custom OWASP and CWE mappings
- Extended with custom detection patterns
- Enhanced with custom validation rules

## Project Structure

After setup, your project structure should look like this:

```
your-project/
├── .github/
│   └── workflows/
│       └── security-bot.yml    # GitHub Actions workflow
├── security-bot/              # Bot directory
│   ├── config/
│   │   └── securitybot-config.yml
│   ├── src/
│   │   ├── index.js          # Main bot implementation
│   │   ├── config.js         # Configuration loader
│   │   └── logger.js         # Logging utility
│   ├── .env                  # Environment variables (not committed)
│   └── package.json         # Dependencies
└── ... (your other project files)
```

## Contributing

1. Fork the repository
2. Create your feature branch (`git checkout -b feature/amazing-feature`)
3. Commit your changes (`git commit -m 'Add some amazing feature'`)
4. Push to the branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

## License

This project is licensed under the MIT License - see the LICENSE file for details.
