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


-----------

```
image: atlassian/pipelines-awscli:latest

stages:
  - validate
  - change
  - deploy

validate_job:
  stage: validate
  variables:
    REGION: ap-northeast-1
  script:
    - aws configure set default.region $REGION
    - export validate_var="aws cloudformation validate-template --template-body=file://$PWD/cfn.yaml"
    - $validate_var
  only:
    - master

change_job:
  stage: change
  variables:
    STACK_NAME: iot-policy
    REGION: ap-northeast-1
  script:
    - aws configure set default.region $REGION
    - export CHANGE_SET=$STACK_NAME-$(date +%Y%m%d%H%M%S)
    - export count=$(aws cloudformation list-stacks | jq -r --arg STACK_NAME "$STACK_NAME" '[.StackSummaries[] | select(.StackName==$STACK_NAME and (.StackStatus=="UPDATE_COMPLETE" or .StackStatus=="CREATE_COMPLETE" or .StackStatus=="UPDATE_ROLLBACK_COMPLETE"))] | length')
    - export create_var="aws cloudformation create-change-set --stack-name $STACK_NAME --change-set-name=$CHANGE_SET --template-body=file://$PWD/cfn.yaml --capabilities CAPABILITY_NAMED_IAM"
    - export describe_var="aws cloudformation describe-change-set --stack-name $STACK_NAME --change-set-name=$CHANGE_SET"
    - export execute_var="aws cloudformation execute-change-set --stack-name $STACK_NAME --change-set-name $CHANGE_SET"
    - if (("$count" > "0")); then echo "stack exists, applying change"; $create_var; sleep 3; $describe_var; $execute_var; exit 0; else echo "stack does not exist"; exit 0; fi

deploy_job:
  stage: deploy
  variables:
    STACK_NAME: iot-policy
    REGION: ap-northeast-1
  script:
    - aws configure set default.region $REGION
    - export count=$(aws cloudformation list-stacks | jq -r --arg STACK_NAME "$STACK_NAME" '[.StackSummaries[] | select(.StackName==$STACK_NAME and (.StackStatus=="UPDATE_COMPLETE" or .StackStatus=="CREATE_COMPLETE" or .StackStatus=="UPDATE_ROLLBACK_COMPLETE"))] | length')
    - export deploy_var="aws cloudformation deploy --template-file ./cfn.yaml --stack-name $STACK_NAME --capabilities CAPABILITY_NAMED_IAM"
    - if (("$count" > "0")); then echo "stack exists, skipping deployment"; exit 0; else echo "stack does not exist"; $deploy_var; fi
  only:
    - master

```

```
image: atlassian/pipelines-awscli:latest

stages:
  - validate
  - deploy

validate_job:
  stage: validate
  variables:
    REGION: ap-northeast-1
  script:
    - aws configure set default.region $REGION
    - export validate_var="aws cloudformation validate-template --template-body=file://$PWD/cfn.yaml"
    - $validate_var
  only:
    - master

deploy_job:
  stage: deploy
  variables:
    STACK_NAME: iot-policy
    REGION: ap-northeast-1
  script:
    - aws configure set default.region $REGION
    - export CHANGE_SET=$STACK_NAME-$(date +%Y%m%d%H%M%S)
    - export deploy_var="aws cloudformation deploy --template-file ./cfn.yaml --stack-name $STACK_NAME --capabilities CAPABILITY_NAMED_IAM"
    - export create_var="aws cloudformation create-change-set --stack-name $STACK_NAME --change-set-name=$CHANGE_SET --template-body=file://$PWD/cfn.yaml --capabilities CAPABILITY_NAMED_IAM"
    - export describe_var="aws cloudformation describe-change-set --stack-name $STACK_NAME --change-set-name=$CHANGE_SET"
    - export execute_var="aws cloudformation execute-change-set --stack-name $STACK_NAME --change-set-name $CHANGE_SET"
    - export count=$(aws cloudformation list-stacks | jq -r --arg STACK_NAME "$STACK_NAME" '[.StackSummaries[] | select(.StackName==$STACK_NAME and (.StackStatus=="UPDATE_COMPLETE" or .StackStatus=="CREATE_COMPLETE" or .StackStatus=="UPDATE_ROLLBACK_COMPLETE"))] | length')
    - if (("$count" > "0")); then echo "stack exists"; $create_var; $describe_var; sleep 3; $describe_var; $execute_var; else echo "stack does not exist"; $deploy_var; fi
  only:
    - master


```