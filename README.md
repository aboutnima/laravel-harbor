# Veyoze - Beta version ⚡️ 🛠️

Veyoze is a command-line interface (CLI) tool designed to automate the setup and deployment of preview environments for web applications. It's optimized for creating PR preview environments for Laravel, Nuxt.js, Next.js, and other web applications deployable to Laravel Forge.

## Why Veyoze?

- **Swift Deployment**: Deploy environments swiftly as soon as your PR is ready.
- **Automatic Cleanup**: Ensures no remnants of test environments after PRs are merged or closed.
- **Streamlined Workflow**: Efficiently manage multiple environments, ensuring everyone has their space when needed.

---

## Table of Contents

- [Requirements](#requirements)
- [Getting Started](#getting-started)
- [Configuration](#configuration)
- [Features](#features)
- [Contributing](#contributing)
- [License](#license)
- [Acknowledgments](#acknowledgments)
- [Roadmap](#roadmap)

### Requirements
Before diving in, ensure you have:

- An [app server on Forge](https://forge.laravel.com/docs/servers/types.html#app-servers).
- A Forge [API token](https://forge.laravel.com/docs/1.0/accounts/api.html#create-api-token).

## Getting Started

### Setup and Deploy

To utilize Veyoze CLI on pull requests, integrate a GitHub action workflow that activates on PR events. Begin by creating a workflow named `preview-provision.yml`:

```yml
name: preview-provision
on:
  pull_request:
    types: [opened, edited, reopened, ready_for_review]
jobs:
  veyoze-provision:
    runs-on: ubuntu-latest
    container:
      image: kirschbaumdevelopment/laravel-test-runner:8.1
    steps:
        - name: Install Veyoze
          run: composer global require mehrancodes/veyoze -q
        - name: Start Provisioning
          env:
              FORGE_TOKEN: ${{ secrets.FORGE_TOKEN }}
              FORGE_SERVER: ${{ secrets.FORGE_SERVER }}
              FORGE_GIT_REPOSITORY: ${{ github.repository }}
              FORGE_GIT_BRANCH: ${{ github.head_ref }}
              FORGE_DOMAIN: veyoze.com
          run: veyoze provision
```

This workflow:

1. Listens for specific GitHub action events: opened, edited, reopened, ready_for_review
2. Sets up the workflow container and installs Veyoze.
3. Uses environment keys for Veyoze configuration.
4. Generates a URL like **plt-123-add-new-feature.veyoze.com** upon running the `veyoze provision` command.

### Destroy the Site

To dismantle the site after a PR is merged or closed, introduce a new workflow to execute the `veyoze teardown` command. Create a workflow named `preview-teardown.yml`:

```yml
name: preview-teardown
on:
  pull_request:
    types: [closed]
jobs:
  veyoze-teardown:
    runs-on: ubuntu-latest
    container:
      image: kirschbaumdevelopment/laravel-test-runner:8.1
    steps:
        - name: Install Veyoze
          run: composer global require mehrancodes/veyoze -q
        - name: Start Teardown
          env:
              FORGE_TOKEN: ${{ secrets.FORGE_TOKEN }}
              FORGE_SERVER: ${{ secrets.FORGE_SERVER }}
              FORGE_GIT_REPOSITORY: ${{ github.repository }}
              FORGE_GIT_BRANCH: ${{ github.head_ref }}
              FORGE_DOMAIN: veyoze.com
          run: veyoze teardown
```

---

# Configuration

Veyoze utilizes environment keys similar to a Laravel application, offering flexibility and the potential for powerful extensions over time.

#### `FORGE_TOKEN` (required)
This key holds your Forge API token, enabling Veyoze to communicate with Laravel Forge for site creation and resource management. **Always store this value as an encrypted secret; avoid pasting it directly into your workflow file.**

#### `FORGE_SERVER` (required)
Specify the server where Veyoze should create and deploy a site.  The value to use here is the Forge "Server ID" for the target server.  Examples: 723019, 68342, etc.

#### `FORGE_GIT_REPOSITORY` (required)
Indicate the Git repository name, such as 'mehrancodes/veyoze'.

#### `FORGE_GIT_BRANCH` (required)
Provide the Git repository branch name, for instance, 'add-new-feature'.

#### `FORGE_DOMAIN` (required)
Define the website's domain name. Acceptable formats include 'veyoze.com' or 'api.veyoze.com'.

#### `FORGE_GIT_PROVIDER`
Identify the Git service provider. Options include GitHub, GitLab, etc., with 'github' as the default. Refer to the [Forge API documentation](https://forge.laravel.com/api-documentation#install-new) for more details.

#### `FORGE_SUBDOMAIN_PATTERN`
Veyoze constructs the site's subdomain based on your branch name. If your branch name follows a format like **YOUR_TEAM_SHORT_NAME-TICKET_ID** (e.g., **apt-123-added-new-feature**), you can employ a [regex pattern](https://en.wikipedia.org/wiki/Regular_expression) to abbreviate your subdomain:

```shell
FORGE_SUBDOMAIN_PATTERN: /^[a-z]{1,3}-(\d{1,4})/i
```

#### `FORGE_DEPLOY_SCRIPT`
By setting the [Forge Deploy Script](https://forge.laravel.com/docs/sites/deployments.html#deploy-script) environment key, you can seamlessly relay your custom bash deploy script to your new Forge site before project deployment. You can provide it directly or via a [GitHub action variable](https://docs.github.com/en/actions/learn-github-actions/variables) for improved workflow readability. Here's a basic deploy script example:

```shell
FORGE_DEPLOY_SCRIPT: "cd $FORGE_SITE_PATH; git pull origin $FORGE_SITE_BRANCH; composer install; etc..."
```

For extensive deploy scripts, consider using an action variable or secret:

```shell
# Store the script in a variable, e.g., LARAVEL_DEPLOY_SCRIPTS:
cd $FORGE_SITE_PATH
git pull origin $FORGE_SITE_BRANCH
composer install

# Then reference it like this:
FORGE_DEPLOY_SCRIPT: ${{ vars.LARAVEL_DEPLOY_SCRIPTS }}
```

#### `FORGE_ENV_KEYS`
Employ this key to introduce or update your project's custom environment keys, separated by ';':

```bash
FORGE_ENV_KEYS="GOOGLE_API=${{secrets.APP_NAME}}; SMS_API=${{secrets.SMS_API}}"
```

Alternatively, as mentioned in the deploy script section, utilize a secret key for easier management:

```shell
FORGE_ENV_KEYS: ${{ secrets.LARAVEL_ENV_KEYS }}
```

#### `FORGE_NGINX_TEMPLATE`
This key allows you to specify a custom [Nginx configuration](https://forge.laravel.com/docs/servers/nginx-templates.html) template. This is useful when you need to tweak the default Nginx settings to cater to specific requirements of your application.

#### `FORGE_NGINX_SUBSTITUTE`
Define key=value pairs to customize the Nginx template. This is particularly handy when you need to automatically set values inside your Nginx template, such as proxying your Nuxt.js app to a specific port. An example value might be: 'NUXT_PORT=1234; NEXT_PORT=7542'.

#### `FORGE_PHP_VERSION`
Specify the desired PHP version for your application. The default is 'php82', but you can set it to other supported versions installed on your server as per your application's requirements.

#### `FORGE_PROJECT_TYPE`
Indicate the [type of the project](https://forge.laravel.com/api-documentation#create-site). The default is 'php', but depending on your application's stack, you might need to specify a different type.

#### `FORGE_SITE_ISOLATION`
A flag to determine if [user isolation](https://forge.laravel.com/docs/sites/user-isolation.html) is required. By default, it's set to false.

#### `FORGE_JOB_SCHEDULER`
This flag indicates whether a job scheduler, like Laravel's task scheduler, is needed. By default, it's set to false.

#### `FORGE_AUTO_SOURCE_REQUIRED`
A flag to determine if environment variables should be auto-sourced during deployment. By default, it's set to false. Enable this if your deployment process requires environment variables to be sourced automatically.

#### `FORGE_DB_CREATION_REQUIRED`
Indicate if a database should be automatically created during the provisioning process. By default, it's set to false.

#### `FORGE_SSL_REQUIRED`
This flag indicates whether SSL certification should be enabled for the site. While the default setting is false, enabling this ensures your site is served securely over HTTPS.

**Note**: If you enable this, ensure you've added [a wildcard subdomain DNS record](https://en.wikipedia.org/wiki/Wildcard_DNS_record) pointing to your Forge server.

#### `FORGE_QUICK_DEPLOY`
This flag allows you to toggle the [Quick Deploy](https://forge.laravel.com/docs/sites/deployments.html#deploy-script) feature. By default, it's set to false.

**Caution**: If you intend to enable this feature on your site, ensure the provision workflow isn't triggered.

I've made the descriptions more concise and clear, and added emphasis where needed for clarity.

#### `FORGE_WAIT_ON_SSL`
A flag to pause the provisioning process until the SSL setup completes. By default, it's set to true, ensuring that the provisioning doesn't proceed until the SSL is fully set up.

#### `FORGE_WAIT_ON_DEPLOY`
This flag pauses the provisioning process until the site deployment completes. By default, it's true, ensuring a smooth and complete deployment before any subsequent steps.

---

## Features

WIP

## Contributing

We welcome contributions! Please see our [CONTRIBUTING.md](https://github.com/mehrancodes/veyoze/blob/main/CONTRIBUTING.md) for details on how to contribute and the process for submitting pull requests.

## License

This project is licensed under the MIT License - see the [LICENSE.md](https://github.com/mehrancodes/veyoze/blob/main/LICENSE.md) file for details.


## Acknowledgments

**Q: Is it secure to use sensitive data like API tokens in our workflow?**

**A:** Yes, when passed securely. We use [GitHub actions secrets](https://docs.github.com/en/actions/security-guides/using-secrets-in-github-actions) to safely pass the Forge token and server ID.

**Q: Can we get a shorter link for long branch names?**

**A:** Absolutely! By configuring the [Subdomain Pattern](https://github.com/mehrancodes/veyoze#forge_subdomain_pattern) you can shorten the domain, e.g., **plt-123.veyoze.com**.

**Q: Why use environment keys for configuration instead of command arguments?**

**A:** We have around 20 configurations and more planned. Using environment keys keeps things organized. However, we're open to suggestions!

**Q: How can we control the conditions under which this workflow runs?**

**A:** Check our [veyoze-laravel-sample workflow](https://github.com/mehrancodes/veyoze-laravel-sample/blob/92fce07b6b63bf665ad2063db7f2ad00fa9f3f31/.github/workflows/pr-preview-provision.yml#L7). It specifies conditions like the PR not being a draft or the PR title containing `[veyoze]`.

**Q: Is there a standalone build for Veyoze CLI?**

**A:** Yes! Grab the latest build [from the releases page](https://github.com/mehrancodes/veyoze/releases). It's quicker to set up and highly recommended.


Certainly! Including a "To-Do" or "Roadmap" section in your README is a great way to communicate future plans and improvements to your users and potential contributors. Here's how you can incorporate that into the template:

---

## Roadmap

As this CLI tool has been crafted as an MVP and has undergone real-world testing, we're now focusing on enhancing its robustness and expanding its capabilities. Here's what's on our radar:

- 🚀 **Preview Link Comments**: Plan to automatically comment the preview link on the pull request once provisioning is complete.

- 🛠️ **Enhanced Error Handling**: Aiming for comprehensive error handling throughout the entire CLI process to ensure smooth user experiences.

- 📣 **Slack Integration**: Integrate with Slack to provide real-time notifications and updates.

- 📘 **Expanded Documentation**: We'll be adding more examples showcasing the tool's versatility, such as:
    - Using Laravel as an API backend with Nuxt.js on the frontend for SSR handling.
    - Using with projects like WordPress and more.

- 🧪 **Testing**: Preparing unit tests and feature tests to ensure the tool's reliability and stability.
