[gh-conf-ssh-keys]: https://docs.github.com/en/authentication/connecting-to-github-with-ssh
[gh-troubleshoot-ssh-keys-conf]: https://docs.github.com/authentication/troubleshooting-ssh
[claude-pricing]: https://claude.com/pricing
[claude-install]: https://claude.com/product/claude-code
[cc-settings]: https://docs.anthropic.com/en/docs/claude-code/settings
[cc-memory]: https://docs.anthropic.com/en/docs/claude-code/memory
[cc-permissions]: https://docs.anthropic.com/en/docs/claude-code/settings#permissions
[cc-sandboxing]: https://www.anthropic.com/engineering/claude-code-sandboxing
[owasp-wstg]: https://owasp.org/www-project-web-security-testing-guide/
[owasp-asvs]: https://owasp.org/www-project-application-security-verification-standard/

![AI Assisted Secure Coding](resources/heading-image.png)

# Prerequisites

- GitHub account, a free account works.
- VS Code (extensions may vary)
- Your preferred terminal emulator (optional, only needed if running locally)
- Claude Subscription, a _Free_ plan works. You can get one at [Claude Pricing][claude-pricing]

> **Note (local setup only):** If you plan to clone the repository using SSH, make sure to configure your [SSH Keys][gh-conf-ssh-keys] beforehand. Refer to the [troubleshooting guide][gh-troubleshoot-ssh-keys-conf] if you run into issues.

# Setup

We'll go over two different exercises where we'll work together with Claude Code to perform quality-related tasks; the exercises can be carried out locally on your machine or in _GitHub Codespaces_.

## 1. Fork the Repository

You'll work on your own fork of this repository. To get started, locate the fork button at the top of the repository page, next to _Watch_ and _Star_.

![gh-code-toolbar](resources/gh-repo-toolbar.png)

1. Click the Fork button.
2. Make sure your user is selected in the _Owner_ drop-down.
3. Provide a name for the repository or leave as-is.

You may now browse to the new repository under your account.

## 2. Clone or Open the Repository

### Option 1 - Start Codespaces

Once you are on the front page of your forked repository:

1. Click the _Code_ green button which will open up a palette.
2. Select _Codespaces_ and then click on the green button _Create Codespace on main_.

![gh-repo-code](resources/gh-codespaces.png)

This will open a new tab where your codespace will be bootstrapped — wait for it to finish, and you should then see an editor with the file tree on the left side.

### Option 2 - Clone the Repository

Once you are on the front page of your forked repository:

1. Click on the _Code_ green button which will open a palette.
2. While on the _Local_ tab, _Clone_ using SSH; you may alternatively use the GitHub CLI if you have already set it up.

## 3. Installing Claude Code

Navigate to [Install Claude Code][claude-install] and select the option that suits your platform and workstation configuration.

Once installed, run the following to confirm everything is working:

```bash
claude --version
```

### GitHub Codespaces

Once your codespace is up and running, install the Claude Code extension:

1. Open the Extensions sidebar (_Ctrl+Shift+X_).
2. Search for **Claude Code**.
3. Click **Install**.

## 4. Authenticating

Start Claude Code by:

- _VS Code or Codespaces_: if the tab isn't visible yet, hit _Ctrl+Shift+P_ to show the command palette, then Claude > Open in New Tab.
- _Terminal emulator_: first make sure you are in the directory where you've cloned the repository, then run the command _claude_.

Select the option to authenticate with a Claude subscription and use your account to sign in.

# Exercises

The following exercises are developed around a fictitious patient journal system which we are going to co-develop with Claude from the ground up, from requirements to development and testing.

Under each application directory for both examples, there is a *prompts* directory with example of prompts for Claude to carry out the tasks.

## E1 - Procure Security Requirements

The file `patient_journal_api/README.md` contains a basic description of the application, functional requirements, details of the planned infrastructure and it stack. The goal is to have Claude read the app description and complement it with security requirements considerations, using [OWASP ASVS][owasp-asvs] as the base/source catalog of security requirements.

Generate an initial threat model using the STRIDE method for the application. 


## E2 - Address Weaknesses in Application Code, CI/CD and Application Infrastructure

We have a second application in `malware_analysis_api` with an existing implementation that we need to take a look at to:

- Find weaknesses and vulnerabilities in the existing application code
- Review security configuration in the CI/CD pipelines
- Review the terraform project to get suggestions
- Add security testing according to [OWASP Web Security Testing Guide][owasp-wstg]

Ensure this runs as *Plan Mode*.
