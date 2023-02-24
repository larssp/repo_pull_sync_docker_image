# GitLab Repo Mirror Pull Sync - Docker Image + GitLab Scheduled Pipeline

Useful Docker image that can be used in combination with a [GitLab Scheduled Pipelines](https://docs.gitlab.com/ee/ci/pipelines/schedules.html) for pulling remote repositories.

Normally, GitLab Community Edition (GitLab CE) _only_ offers you to mirroring repositories via `push` direction. This Docker image with yaml file + scheduler allows you to sync a repository via **`pull` mirror direction**!

Have the "GitLab mirror pull" feature for free, even when using GitLab CE! Use the steps below to get started.

## Docker Image

We are following Docker image in our pipeline: [danger89/repo_mirror_pull](https://hub.docker.com/r/danger89/repo_mirror_pull) ([source](./Dockerfile))

## GitLab Pipeline

[GitLab Scheduled Pipeline](https://docs.gitlab.com/ee/ci/pipelines/schedules.html) yaml file (`.gitlab-ci.yml`). No changes are needed, the variables can be changed in the GitLab Schedule (see next heading).

Usually you start with an empty repo. Create the first branch containing the `.gitlab-ci.yml` file with a different name than the one you want to mirror, e.g. 'repo_pull_sync'.

If the rebase fails the pipeline aborts automatically (so no git push).

```yml
repo_pull_sync:
  image: danger89/repo_mirror_pull:latest
  rules:
    - if: '$CI_PIPELINE_SOURCE == "schedule"'
    - if: $REMOTE_URL
    - if: $REMOTE_BRANCH
    - if: $SSH_DEPLOY_KEY
  before_script:
    - git config --global user.name "${GITLAB_USER_NAME}"
    - git config --global user.email "${GITLAB_USER_EMAIL}"
    - mkdir ~/.ssh
    - cp ${CI_KNOWN_HOSTS} ~/.ssh/known_hosts
    - cp ${SSH_DEPLOY_KEY} ~/.ssh/id_rsa
    - chmod 600 ~/.ssh/id_rsa
    - git remote remove ssh_origin || true
    - git remote add ssh_origin "git@$CI_SERVER_HOST:$CI_PROJECT_PATH.git" 
    - git remote remove upstream || true
    - git remote add upstream $REMOTE_URL
  script:
    - git fetch
    - git fetch upstream
    - git checkout $REMOTE_BRANCH || git checkout --track upstream/$REMOTE_BRANCH
    - git pull
    - git rebase upstream/$REMOTE_BRANCH
    - git push ssh_origin
```

*Note:* If you want to use `git merge` instead of `git rebase` that is up to you. You can change the script above to your needs.

## Create GitLab Schedule

1. Create Project Deploy key fist:

   * Generate an RSA SSH key pair with `ssh-keygen -m pem`
   * Go to: `Settings -> Repository -> Deploy keys` and add the public key. Check 'Grant write permissions to this key'. 

2. Get the ssh fingerprints of your gitlab instance

   * `ssh-keyscan <gitlab instance URL> > fingerprints.txt`

3. Create a new GitLab Schedule:

* Go to: `CI/CD -> Schedules -> New schedule`. 
  * With the following 2 'Variable' variables:
    * `REMOTE_URL` (example: `https://github.com/project/repo.git`)
    * `REMOTE_BRANCH` (example: `master`)
  * With the following 2 'File' variables:
    * `SSH_DEPLOY_KEY` (copy the private ssh key here)
    * `CI_KNOWN_HOSTS` (copy contents of fingerprints.txt)
* Save pipeline schedule
* Press the Play button to trigger the Schedule prematurely (for testing purpose)
