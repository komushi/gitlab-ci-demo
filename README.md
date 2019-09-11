# GitLab CI DEMO for CloudFormation

## 0. Prerequisite
* A GitLab Server with GitLab Runner.
* An AWS IAM_USER with acess to call CloudFormation API.
* In the project's Settings's CI/CD AWS_ACCESS_KEY_ID & AWS_SECRET_ACCESS_KEY are added to varialbles.

## 1. Add .gitlab-ci.yml

### 1-1. Attribute "images"
* Use aws-cli enabled docker image
```
image: atlassian/pipelines-awscli:latest
```

### 1-2. Attribute "stages"

* Jobs of the same stage are run in parallel.
* Jobs of the next stage are run after the jobs from the previous stage complete successfully.

### stages example
```
stages:
  - build
  - test
  - deploy
```

* First, all jobs of build are executed in parallel.
* If all jobs of build succeed, the test jobs are executed in parallel.
* If all jobs of test succeed, the deploy jobs are executed in parallel.
* If all jobs of deploy succeed, the commit is marked as passed.
* If any of the previous jobs fails, the commit is marked as failed and no jobs of further stage are executed.

### 1-3. Attributes under "job"
* "stage" to define a stage to invoke the job.
* "script" to define the shell commands.
* "only / except" to to limit when jobs are created.

### 1-4. CloudFormation cli commands used in "script"
* aws cloudformation validate-template
* aws cloudformation list-stacks
* aws cloudformation deploy
* aws cloudformation create-change-set
* aws cloudformation describe-change-set
* aws cloudformation execute-change-set

## 2. Using GitLab CI
### 2-1. Edit cfn.yaml and add/commit/push.
```
$ git add .
$ git commit -m 'message'
$ git push
```

### Options to Skip GitLab CI
* use "git push -o ci.skip"
* use "except" in .gitlab-ci.yml


### 2-2. Check the CI/CD page on GitLab

# References:
https://docs.gitlab.com/ee/ci/quick_start/README.html
https://docs.gitlab.com/ee/ci/yaml/README.html
https://docs.aws.amazon.com/cli/latest/reference/cloudformation/index.html#cli-aws-cloudformation
https://qiita.com/diggy-mo/items/7993e79856473a71ac81
