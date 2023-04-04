# Terminus Secrets Manager Plugin

Pantheon’s Secrets Manager Terminus plugin is key to maintaining industry best practices for secure builds and application implementation. Secrets Manager provides a convenient mechanism for you to manage your secrets and API keys directly on the Pantheon platform.

## Overview

### Key Features

- Securely host and maintain secrets on Pantheon

- Use private repositories in Integrated Composer builds

- Create and update secrets via Terminus

- Ability to set a `COMPOSER_AUTH` environment variable and/or a `Composer auth.json` authentication file with Terminus commands

- Ability to define site and org ownership of secrets

- Propegate organization-owned secrets to all the sites in the org

- Ability to define the degree of secrecy for each managed item

- Secrets are encrypted at rest

### Early Access

The Secrets Manager plugin is available for Early Access participants. Features for Secrets Manager are in active development. Pantheon's development team is rolling out new functionality often while this product is in Early Access. Visit the [Pantheon Slack channel](https://slackin.pantheon.io/) (or sign up for the channel if you don't already have an account) to learn how you can enroll in our Early Access program. Please review [Pantheon's Software Evaluation Licensing Terms](https://legal.pantheon.io/#contract-hkqlbwpxo) for more information about access to our software.

## Concepts

### Secret

A key-value pair that should not be exposed to the general public, typically something like a password, API key, or the contents of a peer-to-peer cryptographic certificiate. SSL certificates that your site uses to serve pages are out of scope of this process and are managed by the dashboard in a different place. See the documentation for SSL certificate for details.

### Secret type

This is a field on the secret record. It defines the usage for this secret and how it is consumed. Current types are:

- `runtime`: this secret will be used to retrieve it in application runtime using API calls to the secret service. More info on this to come at a later stage of the Secrets project. This will be the recommended way to set stuff like API keys for third-party integrations in your application.

- `env`: this secret will be used to set environment variables in the application runtime. More info on this to come at a later stage of the Secrets project.

- `composer`: this secret type is used for composer authentication to private packages.

- `file`: this type allows you to store files in the secrets. More info on this to come at a later stage of the Secrets project.

Note that you can only set one type per secret and this cannot be changed later (unless you delete and recreate the secret).

### Secret scope

This is a field on the secret record. It defines the components that have access to the secret value. Current scopes are:

- `ic`: this secret will be readable by the Integrated Composer runtime. You should use this scope to get access to your private repositories.

- `web`: this secret will be readable by the application runtime. More info on this to come at a later stage of the Secrets project.

- `user`: this secret will be readable by the user. This scope should be set if you need to retrieve the secret value at a later stage.

- `ops`: behavior to be defined. More info on this to come at a later stage of the Secrets project.

Note that you can set multiple scopes per secret and they cannot be changed later (unless you delete and recreate the secret).

### Owning entity

Secrets are currently either owned by a site or an organization. Within that owning entity, the secret may have zero or more environment overrides.

### Site-owned secrets

This is a secret that is set for a specific site using the site id. Based on the type and scope, this secret will be loaded on the different scenarios that will be supported by Secrets in Pantheon.

### Organization-owned secrets

This is a secret that is set not for a given site but for an organization. This secret will be inherited by ALL of the sites that are OWNED by this organization. Please note that a [Supporting Organization](https://docs.pantheon.io/agency-tips#become-a-supporting-organization) won't inherit its secrets to the sites, only the Owner organization.

### Environment override

In some cases it will be necessary to have different values for the secret when that secret is accessed in different pantheon environments. You may set an environment override value for any existing secret value. If the secret does not exist, it may not be overriden in any environment and you will get an error trying to set an environment override.

## The life of a secret

When a given runtime (e.g. Integrated Composer runtime or the application runtime) fetches secrets for a given site (and env), it will go like this:

[- NOTE - GRAPH DECISION TREE BEFORE GA - ]

- Fetch secrets for site (of the given type and within the given scopes)

- Apply environment overrides (if any). More info on this to come soon.

- If the site is owned by an organization:

    - Get the organization secrets

    - Apply environment overrides (if any).

    - Merge the organization secrets with the site secrets

Let's go through this with an example: assume you have a site named `my-site` which belongs to an organization `my-org`. You also have another site `my-other-site` which belongs to your personal Pantheon account.

When Integrated Composer attempts to get secrets for `my-other-site` it will go like this:
- Get the secrets of scope `ic` for `my-other-site`.
- Apply environment overrides for the current environment (*).
- Look at `my-other-site` owner. In this case, it is NOT an organization so there are no organization secrets to merge.
- Process the resulting secrets to make them available to Composer.

On the other hand, when Integrated Composer attempts to get secrets for `my-site`, it will go like this:
- Get the secrets of scope `ic` for `my-site`.
- Apply environment overrides for the current environment (see **Note** below).
- Look at the site owner. It determines it is the organization `my-org`.
- Get the secrets for the organization `my-org` with scope `ic`.
- Apply the environment overrides to those secrets for the current environment (see **Note** below).
- Merge the resulting organization secrets with the site secrets with the following caveats:
    - Site secrets take precedence over organization secrets: this mean that the value for site-owned secret named `foo` will be used instead of the value for an org-owned secret with the same name `foo`
    - Only the secrets for the OWNER organization are being merged. If the site has a Supporting Organization, it will be ignored.
- Process the resulting secrets to make them available to Composer.

**Note:** Due to platform design, the "environment" for Integrated Composer will always be either `dev` or a multidev. It will never be `test` or `live` so we don't recommend using "environment" overrides for composer access. The primary use-case for environment overrides is for the CMS key-values and environment variables that need to be different between your production and non-production environments.

## Plugin Usage

### Secrets Manager Plugin Requirements

Secrets Manager requires the following:

- A Pantheon account
- A site that uses [Integrated Composer](https://docs.pantheon.io/guides/integrated-composer) and runs PHP >= 8.0
- Terminus 3

### Installation

Terminus 3.x has built in plugin management.

Run the command below to install Terminus Secrets Manager.

```
terminus self:plugin:install terminus-secrets-manager-plugin
```

### Site secrets Commands

#### Set a secret

The secrets `set` command takes the following format:

- `Name`
- `Value`
- `Type`
- `One or more scopes`


Run the command below to set a secret in Terminus:

```
terminus secret:site:set <site> <secret-name> <secret-value>

[notice] Success

```

```
terminus secret:site:set <site> file.json "{}" --type=file

[notice] Success

```

```
terminus secret:site:set <site> <secret-name> --scope=user,ic

[notice] Success

```

Note: If you do not include a `type` or `scope` flag, their defaults will be `runtime` and `user` respectively.


#### List secrets

The secrets `list` command provides a list of all secrets available for a site. The following fields are available:

- `Name`
- `Scope`
- `Type`
- `Value`
- `Environment Override Values`
- `Org Values`

Note that the `value` field will contain a placeholder value unless the `user` scope was specified when the secret was set.

Run the command below to list a site’s secrets:

`terminus secret:site:list`

```
terminus secret:site:list <site>

 ------------- ------------- ---------------------------
  Secret name   Secret type   Secret value
 ------------- ------------- ---------------------------
  secret-name   env           secrets-content
 ------------- ------------- ---------------------------
```

`terminus secret:site:list`

```
terminus secret:site:list <site> --fields="*"

 ---------------- ------------- ------------------------------------------ --------------- ----------------------------- --------------------
  Secret name      Secret type   Secret value                               Secret scopes   Environment override values   Org values
 ---------------- ------------- ------------------------------------------ --------------- ----------------------------- --------------------
  foo              env           ***                                        web, user
  foo2             runtime       bar2                                       web, user                                     default=barorg
  foo3             env           dummykey                                   web, user       live=sendgrid-live
 ---------------- ------------- ------------------------------------------ --------------- ----------------------------- --------------------
 ```

#### Delete a secret

The secrets `delete` command will remove a secret and all of its overrides.

Run the command below to delete a secret:

```
terminus secret:site:delete <site> <secret-name>

[notice] Success

```

### Organization secrets Commands

#### Set a secret

The secrets `set` command takes the following format:

- `Name`
- `Value`
- `Type`
- `One or more scopes`

Run the command below to set a secret in Terminus:

```
terminus secret:org:set <org> <secret-name> <secret-value>

[notice] Success

```

```
terminus secret:org:set <org> file.json "{}" --type=file

[notice] Success

```

```
terminus secret:org:set <org> <secret-name> --scope=user,ic

[notice] Success

```

Note: If you do not include a `type` or `scope` flag, their defaults will be `runtime` and `user` respectively.


#### List secrets

The secrets `list` command provides a list of all secrets available for an organization. The following fields are available:

- `Name`
- `Scope`
- `Type`
- `Value`
- `Environment Override Values`

Note that the `value` field will contain a placeholder value unless the `user` scope was specified when the secret was set.

Run the command below to list a site’s secrets:

`terminus secret:org:list`

```
terminus secret:org:list <org>

 ------------- ------------- ---------------------------
  Secret name   Secret type   Secret value
 ------------- ------------- ---------------------------
  secret-name   env           secrets-content
 ------------- ------------- ---------------------------
```

`terminus secret:org:list`

```
terminus secret:org:list <org> --fields="*"

 ---------------- ------------- ------------------------------------------ --------------- -----------------------------
  Secret name      Secret type   Secret value                               Secret scopes   Environment override values
 ---------------- ------------- ------------------------------------------ --------------- -----------------------------
  foo              env           bar                                        web, user
  foo2             runtime       bar2                                       web, user
  foo3             env           dummykey                                   web, user       live=sendgrid-live
 ---------------- ------------- ------------------------------------------ --------------- -----------------------------
 ```

#### Delete a secret

The secrets `delete` command will remove a secret and all of its overrides.

Run the command below to delete a secret:

```
terminus secret:org:delete <org> <secret-name>

[notice] Success

```

### Help

Run `terminus list secret` for a complete list of available commands. Use terminus help <command> to get help with a specific command.

## Use Secrets with Integrated Composer

You must configure your private repository and provide an authentication token before you can use the Secrets Manager Terminus plugin with Integrated Composer. You could use either of the following mechanisms to setup this authentication.


### Mechanism 1: Oauth Composer authentication

#### GitHub Repository

1. [Generate a Github token](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token). The Github token must have all "repo" permissions selected.

    NOTE: Check the repo box that selects all child boxes. **Do not** check all child boxes individually as this does not set the correct permissions.

    ![image](https://user-images.githubusercontent.com/87093053/191616923-67732035-08aa-41c3-9a69-4d954ca02560.png) 

1. Set the secret value to the token via terminus: `terminus secret:site:set <site> github-oauth.github.com <github_token> --type=composer --scope=user,ic`

1. Add your private repository to the `repositories` section of `composer.json`:

    ```json
    {
        "type": "vcs",
        "url": "https://github.com/your-organization/your-repository-name"
    }
    ```

    Your repository should contain a `composer.json` that declares a package name in its `name` field. If it is a WordPress plugin or a Drupal module, it should specify a `type` of `wordpress-plugin` or `drupal-module` respectively. For these instructions, we will assume your package name is `your-organization/your-package-name`.

1. Require the package defined by your private repository's `composer.json` by either adding a new record to the `require` section of the site's `composer.json` or with a `composer require` command:

    ```bash
    composer require your-organization/your-package-name
    ```

1. Commit your changes and push to Pantheon.

#### GitLab Repository

1. [Generate a GitLab token](https://docs.gitlab.com/ee/user/profile/personal_access_tokens.html). Ensure that `read_repository` scope is selected for the token.

1. Set the secret value to the token via Terminus: `terminus secret:site:set <site> gitlab-oauth.gitlab.com <gitlab_token> --type=composer --scope=user,ic`

1. Add your private repository to the `repositories` section of `composer.json`:

    ```json
    {
        "type": "vcs",
        "url": "https://gitlab.com/your-group/your-repository-name"
    }
    ```

    Your repository should contain a `composer.json` that declares a package name in its `name` field. If it is a WordPress plugin or a Drupal module, it should specify a `type` of `wordpress-plugin` or `drupal-module` respectively. For these instructions, we will assume your package name is `your-organization/your-package-name`.

1. Require the package defined by your private repository's `composer.json` by either adding a new record to the `require` section of the site's `composer.json` or with a `composer require` command:

    ```bash
    composer require your-group/your-package-name
    ```

1. Commit your changes and push to Pantheon.

#### Bitbucket Repository

1. [Generate a Bitbucket oauth consumer](https://support.atlassian.com/bitbucket-cloud/docs/use-oauth-on-bitbucket-cloud/). Ensure that Read repositories permission is selected for the consumer. Also, set the consumer as private and put a (dummy) callback URL.

1. Set the secret value to the consumer info via Terminus: `terminus secret:site:set <site> bitbucket-oauth.bitbucket.org "<consumer_key> <consumer_secret>" --type=composer --scope=user,ic`

1. Add your private repository to the `repositories` section of `composer.json`:

    ```json
    {
        "type": "vcs",
        "url": "https://bitbucket.org/your-organization/your-repository-name"
    }
    ```

    Your repository should contain a `composer.json` that declares a package name in its `name` field. If it is a WordPress plugin or a Drupal module, it should specify a `type` of `wordpress-plugin` or `drupal-module` respectively. For these instructions, we will assume your package name is `your-organization/your-package-name`.

1. Require the package defined by your private repository's `composer.json` by either adding a new record to the `require` section of the site's `composer.json` or with a `composer require` command:

    ```bash
    composer require your-organization/your-package-name
    ```

1. Commit your changes and push to Pantheon.

### Mechanism 2: HTTP Basic Authentication

You may create a `COMPOSER_AUTH json` and make it available via the `COMPOSER_AUTH` environment variable if you have multiple private repositories on multiple private domains.

Composer has the ability to read private repository access information from the environment variable: `COMPOSER_AUTH`. The `COMPOSER_AUTH` variables must be in a [specific JSON format](https://getcomposer.org/doc/articles/authentication-for-private-packages.md#http-basic). 

Format example:

```bash
#!/bin/bash

read -e COMPOSER_AUTH_JSON <<< {
    "http-basic": {
        "github.com": {
            "username": "my-username1",
            "password": "my-secret-password1"
        },
        "repo.example2.org": {
            "username": "my-username2",
            "password": "my-secret-password2"
        },
        "private.packagist.org": {
            "username": "my-username2",
            "password": "my-secret-password2"
        }
    }
}
EOF

`terminus secret:site:set ${SITE_NAME} COMPOSER_AUTH ${COMPOSER_AUTH_JSON} --type=env --scope=user,ic`
```
