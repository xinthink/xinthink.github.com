---
layout: post
title: "Build your own (GitHub) actions"
subtitle: Using TypeScript & Yarn
feature: /images/feature-actions-telegram.jpg
category: [github-actions, typescript, yarn, telegram]
tags:
path: _posts/typescript/2019-11-07-build-your-own-github-actions.md
excerpt: GitHub Actions, the CI/CD service provided by GitHub, with which you can create custom software development life cycle (SDLC) workflows directly in your GitHub repository. In this article, I share the approach of building custom actions to fit your unique workflow.
---

![GitHub Actions + Telegram](/images/feature-actions-telegram.jpg)

You might probably have heard about [GitHub Actions], the [CI/CD] service provided by GitHub, which is in beta at the time of writing. With which you can create custom software development life cycle (SDLC) workflows directly in your GitHub repository.

An [action] is an individual task or step in workflows. Besides existed actions available on the [Marketplace], which you can adopt right away, in this article, I share the approach of building custom actions in order to fit your unique workflow.

---

As an example, we're going to build an action broadcasting CI/CD results via [Telegram] messages.

![Telegram message screenshot](/images/actions-telegram-screenshot.jpg)

What we're going to build is a [JavaScript action], which is supposed to be lighter, faster than a **Docker container action** (see [Types of actions]).

First, create a repo for your action by applying the [TypeScript action template], name it "action-telegram" or whatever you like.

Before diving into the code, it's better to make a plan about how to organize source files & releases in the repo to keep it maintainable in the future.

For now, we have to **check in** the dependencies (the `node_modules` folder) to make the action functioning, GitHub is **NOT** going to install them for us.

For this reason, it may be helpful to think that each action repo has two phases or sections, one for development and another for distribution: we don't check in dependencies alongside source files, but install them during deployment (distribution). In this case, we use branches to separate the two phases/sections:

| Branch/Tag | Description | Development | Distribution |
|--|--|--|
|`master`|where we commit changes|✅|
|`releases/v{version}`|release candidates e.g. `releases/v1.0`||✅|
|`v{version}`|tags referred from workflows e.g. `v1.0`||✅|

---

## Development

### 1. Preparing

Now we can start developing our action. As mentioned in the subtitle, we prefer [TypeScript] & [Yarn]:
- put TypeScript source files under the `src` folder, compiling into the `lib` folder
- keep `yarn.lock` rather than `package-lock.json`

We should have the following `.gitignore` rules on the **master** branch:

```
node_modules/
__tests__/runner/*
package-lock.json
lib/
```

### 2. Managing metadata

Add an `action.yml` file to define the inputs & outputs of your action. You can find detailed documentation [here][metadata-syntax].

```yaml
# action.yml
name: 'Telegram Action'
description: 'Telegram notification for workflow set up with GitHub Actions'
author: <name>
inputs:
  botToken:
    description: 'The Telegram Bot token'
    required: true
  chatId:
    description: 'The target to which the message will be sent, can be a Telegram Channel or Group'
    required: true
  jobStatus:
    description: "The current status of the job: job.status, defaults to 'success'"
    default: 'success'
  skipSuccess:
    description: "Only non-success notifications will be sent if true, include success notifications otherwise, defaults to 'true'"
    default: 'true'
runs:
  using: 'node12'
  main: 'lib/main.js'
```

### 3. Coding

Install dependencies first:
- add [actions toolkit]: `yarn add @actions/github` so that we can access properties of the workflow itself
- and also `yarn add request-promise-native` and `yarn add -D @types/request-promise-native synp`

And now the code, finally! 🤣

```ts
// src/main.ts
import * as core from '@actions/core';
import { context } from '@actions/github';
import * as request from 'request-promise-native';

(async function run() {
  try {
    const botToken = core.getInput('botToken');
    const chatId = core.getInput('chatId');
    const jobStatus = core.getInput('jobStatus');
    const skipSuccess = (core.getInput('skipSuccess') || 'true') === 'true';
    core.debug(`sending message, status=${jobStatus} skipSuccess=${skipSuccess} payload=${JSON.stringify(context.payload)}`);
    await _sendMessage(botToken, chatId, jobStatus, skipSuccess);
    core.debug('message sent');
  } catch (error) {
    core.setFailed(error.message);
  }
})()

/**
 * Send a Telegram message.
 * @param botToken the Telegram bot token to send the message
 * @param chatId id of targeted channel or group, to which the message will be sent
 * @param jobStatus status of the job
 */
async function _sendMessage(
  botToken: String,
  chatId: String,
  jobStatus: String = 'success',
  skipSuccess: Boolean = true,
) {
  const status = (jobStatus || '').toLowerCase();
  if (status === 'success' && skipSuccess) {
    core.debug('skipping successful job');
    return;
  }

  const { repo, ref, sha, workflow, actor } = context;
  const repoFullname = `${repo.owner}/${repo.repo}`;
  const repoUrl = `https://github.com/${repoFullname}`;
  let icon: String;
  switch (status) {
    case 'success': icon = '✅'; break;
    case 'failure': icon = '🔴'; break;
    default: icon = '⚠️'; break;
  }
  const uri = `https://api.telegram.org/bot${botToken}/sendMessage`;
  const text = `${icon} [${repoFullname}](${repoUrl}/actions) ${workflow} *${jobStatus}*

  \`${ref}\` \`${sha.substr(0, 7)}\` by *${actor}*

  [View details](${repoUrl}/commit/${sha}/checks)`;
  return request.post(uri, {
    body: {
      text,
      chat_id: chatId,
      parse_mode: 'Markdown',
    },
    json: true,
  });
}
```

---

## Distribution

As previously explained, the word *distribution* here means the process of publishing your action. You don't have to deploy it anywhere, GitHub will handle it behind the scenes.

### 1. Preparing

Create a branch named `releases/v1.0` assuming it's your first release, and you can find detailed versioning guides [here][versioning].

Append the following rules to `.gitignore`, as we have to commit all the **runtime** stuff (dependencies & compiled code) to **release** branches.

```
# for release branches only
!lib/
!package-lock.json
!node_modules/
```

### 2. Build & Publish

Before launching the first release, we have many works to do, including compiling the code and packaging production modules. Let's put down the procedures to scripts, making them reusable for future releases.

```json
{
  "scripts": {
    "build": "tsc",
    "clean": "rm -rf lib package-lock.json",
    "rebuild": "yarn clean && yarn build",
    "release": "yarn clean && yarn && yarn build && yarn npm-lock && mv -f package-lock.json npm.lock && rm -rf node_modules && yarn --prod && mv npm.lock package-lock.json",
    "dev": "yarn clean && yarn",
    "npm-lock": "synp -f -s yarn.lock"
  }
}
```

It's simpler than what it looks like, explained as followings:
- `yarn release`: run it before committing to release branches:
  - It compiles the code using `tsc` and generates a `package-lock.json` file via `synp` (both are dev dependencies),
  - and then cleans up to keep production packages only by `yarn install --prod`
- `yarn dev`: run it before making changes on the **master** branch

Now tag and push your commits to your remote repo (see [versioning][toolkit-versioning]), and your action is published! 🎉

---

## Validation & Usage

To make sure your first action is functioning properly, add a step to any workflow of any repo (the same action repo will also work). To get your own [Telegram Bot token] & `chatId`, you may find these [instructions][get-bot-token] helpful (ignore the *local.ts* part).

```yaml
# .github/workflows/check.yml
- name: notification
  uses: <user>/<repo>@v1 # use relative path if this's the same repo of your action
  with:
    botToken: <TelegramBotToken>
    chatId: <TelegramChatId>
    jobStatus: ${{ "{{ job.status " }}}}
```

Trigger the workflow by making a few commits to the repo, to find out if everything works fine.

### Tips & Best Practices (in my opinion)

- Make release branches **read-only**: Make changes on **master** branch only, and then merge them into release branches, to avoid conflicts
- Use a dedicated branch, e.g., `release-template`, to keep **shared configurations** ascroll all release branches, and use it as a starting point for all your future releases (see my real-world [action repo][release-template])
- Enable diagnostic logging by setting the secret `ACTIONS_STEP_DEBUG` to `true` in your repo (see [Enabling debug logging])

---

In this article, I shared my practices of building & distributing a custom action for [GitHub Actions]. Please let me know if you have any thoughts.

Happy coding!

---

This article is originally posted on [my Medium publication][origin-link].


[CI/CD]: https://en.wikipedia.org/wiki/CI/CD
[GitHub Actions]: https://github.com/features/actions
[Action]: https://help.github.com/en/articles/about-actions
[Marketplace]: https://github.com/marketplace?type=actions
[origin-link]: https://xinthink.com/build-your-own-github-actions-f3454f22f202
[Medium]: https://medium.com
[Telegram]: https://telegram.org
[JavaScript action]: https://help.github.com/en/github/automating-your-workflow-with-github-actions/creating-a-javascript-action
[Types of actions]: https://help.github.com/en/github/automating-your-workflow-with-github-actions/about-actions#types-of-actions
[TypeScript action template]: https://github.com/actions/typescript-action
[TypeScript]: https://www.typescriptlang.org
[Yarn]: https://yarnpkg.com
[metadata-syntax]: https://help.github.com/en/github/automating-your-workflow-with-github-actions/metadata-syntax-for-github-actions
[actions toolkit]: https://help.github.com/en/github/automating-your-workflow-with-github-actions/creating-a-javascript-action#add-actions-toolkit-packages
[versioning]: https://help.github.com/en/github/automating-your-workflow-with-github-actions/about-actions#versioning-your-action
[toolkit-versioning]: https://github.com/actions/toolkit/blob/master/docs/action-versioning.md
[Telegram Bot token]: https://core.telegram.org/bots
[get-bot-token]: https://github.com/xinthink/webhooks#broadcast-travis-ci-results-to-a-telegram-channel
[release-template]: https://github.com/xinthink/action-telegram/tree/release-template
[Enabling debug logging]: https://help.github.com/en/actions/automating-your-workflow-with-github-actions/managing-a-workflow-run#enabling-debug-logging
