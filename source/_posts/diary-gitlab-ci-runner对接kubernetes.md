---
title: "gitlab-ci runner对接kubernetes"
date: 2021-04-29 14:29:46
tags:
- "gitlab"
- "ci"
- "个人日记"
---

# 为什么弄这个？
<!--more-->

gitlab可以提交代码，真正发版时，不是发布提交的分支，而是提交的分支合并至一个发布分支。gitlab-ci可以在合并前进行一系列的测试，如果正常才允许你进行发版操作，如果异常，就不能发版。



# git发布模型

![image-20210429195540017](http://myapp.img.mykernel.cn/image-20210429195540017.png)



# 部署gitrunner

http://blog.mykernel.cn/2021/02/05/Helm%E7%AE%A1%E7%90%86%E5%99%A8/#%E9%83%A8%E7%BD%B2gitlab-runner



# git-runner的配置文件

```bash
image: python:latest

before_script:
  - echo "准备基础环境 before_script"
  - date
  - pwd
  except:
  - master

test:
  script:
    # dev to master merge请求，会扫描dev源分支。并构建
    - echo "测试步骤 only merge_requests"
    - pwd
  only:
    - merge_requests
```

> ```markdown
> only:
> - merge_requests
> ```
>
>   表示 merge请求发起时，会自动调用此piepline
>
> 成功才会merge.

# 配置merge

![image-20210429200023372](http://myapp.img.mykernel.cn/image-20210429200023372.png)

# 在线更新deve分支的代码

![image-20210429200132837](http://myapp.img.mykernel.cn/image-20210429200132837.png)

注意上面出现创建合并请求

# 创建合并请求

![image-20210429200208740](http://myapp.img.mykernel.cn/image-20210429200208740.png)

> 合并成功后是否删除源分支，一般合并之后就可以删除了。

![image-20210429200405437](http://myapp.img.mykernel.cn/image-20210429200405437.png)

# auto devops

默认模板是，gitlab版本 12.5.2-ee，则下面是模板路径

https://gitlab.com/gitlab-org/gitlab/-/blob/v12.5.2-ee/lib/gitlab/ci/templates/

## gitlabrunner

```diff
imagePullPolicy: IfNotPresent
replicas: 1
+gitlabUrl: http://182.148.48.146:9292
+runnerRegistrationToken: "ocoTP9iQ9xben9E-Zk_T"
terminationGracePeriodSeconds: 3600
concurrent: 10
checkInterval: 30
rbac:
  create: true
  rules: 
  - resources: ["pods", "secrets"]
    verbs: ["get", "list", "watch", "create", "patch", "delete"]
  - apiGroups: [""]
    resources: ["pods/exec"]
    verbs: ["create", "patch", "delete"]
  clusterWideAccess: false
  serviceAccountName: default
  podSecurityPolicy:
    enabled: false
    resourceNames:
+    - gitlab-runner
metrics:
  enabled: true
runners:
  config: |
    [[runners]]
+      clone_url = "http://182.148.48.146:9292" # 克隆地址
+      url = "http://182.148.48.146:9292"       
      [runners.kubernetes]
        namespace = "{{.Release.Namespace}}"
        image = "ubuntu:16.04"
+  privileged: true
  cache: {}
  builds: {}
  services: {}
  helpers: {}
  serviceAccountName: default
securityContext:
  runAsUser: 100
  fsGroup: 65533
resources: {}
affinity: {}
nodeSelector: {}
tolerations: []
hostAliases: []
podAnnotations: {}
podLabels: {}
secrets: []
configMaps: {}
```



## 准备仓库

自动生成dockerfile

```dockerfile
# This file is a template, and might need editing before it works on your project.
FROM python:3.6-alpine


WORKDIR /usr/src/app

COPY requirements.txt /usr/src/app/
RUN pip install --no-cache-dir -r requirements.txt

COPY . /usr/src/app


# For some other command
CMD ["python", "app.py"]

```

app.py

```python
print('hello world')
myurl = "http://www.baidu.com"
print(myurl)

```

requirements.txt

```python
```



1. 删除 `.gitlab-ci.yaml`

2. 在k8s节点准备镜像。

   ```bash
   docker pull registry.cn-hangzhou.aliyuncs.com/slck8s/auto-build-image
   docker tag registry.cn-hangzhou.aliyuncs.com/slck8s/auto-build-image:latest registry.gitlab.com/gitlab-org/cluster-integration/auto-build-image/master:stable
   
   docker pull registry.cn-hangzhou.aliyuncs.com/slck8s/auto-build-image:auto-deploy-image
   docker tag registry.cn-hangzhou.aliyuncs.com/slck8s/auto-build-image:auto-deploy-image registry.gitlab.com/gitlab-org/cluster-integration/auto-deploy-image:v2.6.0
   
   
   docker pull registry.cn-hangzhou.aliyuncs.com/slck8s/auto-build-image:gitlab-runner-helper
   docker tag registry.cn-hangzhou.aliyuncs.com/slck8s/auto-build-image:gitlab-runner-helper registry.gitlab.com/gitlab-org/gitlab-runner/gitlab-runner-helper:x86_64-8925d9a0
   
   
   docker pull registry.gitlab.com/gitlab-org/ci-cd/codequality:0.85.24
   docker tag registry.cn-hangzhou.aliyuncs.com/slck8s/auto-build-image:codequality  registry.cn-hangzhou.aliyuncs.com/grasp_base_images/codequality:0.85.24
   docker push registry.cn-hangzhou.aliyuncs.com/grasp_base_images/codequality:0.85.24
   
   ```

3. 配置runner克隆地址，如果gitlab配置的url不行，手工配置gitlab地址。

   安装gitlab-runner时配置

   ```diff
   162 runners:
   163   # runner configuration, where the multi line strings is evaluated as
   164   # template so you can specify helm values inside of it.
   165   #
   166   # tpl: https://helm.sh/docs/howto/charts_tips_and_tricks/#using-the-tpl-function
   167   # runner configuration: https://docs.gitlab.com/runner/configuration/advanced-configuration.html
   168   config: |
   +169     [[runners]]
   +170       clone_url = "http://xxxx:9292"
   +171       url = "http://xxxx:9292"
   172       [runners.kubernetes]
   173         namespace = "{{.Release.Namespace}}"                                                                             
   174         image = "ubuntu:16.04"
   ```

   > 注意 runners 下

   更新配置

   ```bash
   helm upgrade -n gitlab gitlab-runner         --set gitlabUrl=http://1xxx:9292,runnerRegistrationToken=123123         gitlab/gitlab-runner -f gitlab-runner/values.yaml
   ```

   

4. autodevops 使用的模板仓库: https://gitlab.com/gitlab-org/gitlab.git

   模板位置: 

   - build: https://gitlab.com/gitlab-org/gitlab/-/blob/master/lib/gitlab/ci/templates/Auto-DevOps.gitlab-ci.yml
   - https://gitlab.com/gitlab-org/gitlab/blob/master/lib/gitlab/ci/templates/Jobs/Build.gitlab-ci.yml

5. 配置仓库生成镜像的仓库地址和debug模式。[AutoDevops变量](https://docs.gitlab.com/ee/topics/autodevops/customize.html#cicd-variables)

   - CI_DEBUG_TRACE="true"

   - CI_REGISTRY_IMAGE="registry.cn-hangzhou.aliyuncs.com/<名称空间>" 构建镜像的标签，推送的标签。

   - CI_COMMIT_REF_SLUG=“” docker仓库的应用名

   - CI_REGISTRY_USER="" docker仓库 认证用户名

   - CI_REGISTRY_PASSWORD=“” docker仓库 认证密码

   - CI_REGISTRY=“” docker仓库认证的仓库 (生成k8ssecret, 和build时推送的认证仓库)

   - CI_DEPLOY_USER=""  生成k8s的secret，认证镜像仓库的用户

   - CI_REGISTRY=“” 生成k8s的secret，docker仓库认证的仓库

   - 禁用阶段：https://docs.gitlab.com/ee/topics/autodevops/customize.html

     哪个阶段不成功时，可以临时禁用。
     
     ```bash
     CANARY_ENABLED # 启动金丝雀部署
     KUBE_INGRESS_BASE_DOMAIN # 外部访问的域名	
     AUTO_DEVOPS_FORCE_DEPLOY_V2= 1
     BUILD_DISABLED=true
     CI_COMMIT_REF_SLUG=mydemo 镜像名
     CI_DEBUG_TRACE=true ci过程显示更少的日志
     CI_SERVER_HOST=182.148.48.146:9292
     CODE_QUALITY_DISABLED=true
     POSTGRES_ENABLED=false
     TEST_DISABLED=true
     
     #https://docs.gitlab.com/ee/topics/autodevops/customize.html#deploy-policy-for-staging-and-production-environments
     #高可用 至少2个副本
     CANARY_REPLICAS=2
     REPLICAS=2
     
     ```
   
   ![image-20210806100120629](http://myapp.img.mykernel.cn/image-20210806100120629.png)

> 注意：在图片中填的值不需要加引号

## 验证镜像

```bash
docker pull registry.cn-hangzhou.aliyuncs.com/ishop-general/mydemo:latest

root@51a60b42fd47:/data/songliangcheng# docker run --rm -it  registry.cn-hangzhou.aliyuncs.com/ishop-general/mydemo:latest
hello world
http://www.baidu.com
```

证明仓库自动构建的镜像可用



## 更新autodevops的镜像版本

https://docs.gitlab.com/ee/topics/autodevops/upgrading_auto_deploy_dependencies.html#verify-dependency-versions

这个向导解释怎么通过更新或不同的autodeops依赖的版本来升级你的应用。

### 检验使用的版本

- For self-managed instances, the [stable Auto Deploy template bundled with the GitLab package](https://gitlab.com/gitlab-org/gitlab/-/blob/master/lib/gitlab/ci/templates/Jobs/Deploy.gitlab-ci.yml) is being used.

- The GitLab.com stable Auto Deploy template is being used if one of the following is true:

  - Your Auto DevOps project doesn’t have a `.gitlab-ci.yml` file.
  - Your Auto DevOps project has a `.gitlab-ci.yml` and [includes](https://docs.gitlab.com/ee/ci/yaml/index.html#includetemplate) the `Auto-DevOps.gitlab-ci.yml` template.

- The latest Auto Deploy template is being used if both of the following is true:
  - Your Auto DevOps project has a `.gitlab-ci.yml` file and [includes](https://docs.gitlab.com/ee/ci/yaml/index.html#includetemplate) the `Auto-DevOps.gitlab-ci.yml` template.
  - It also includes [the latest Auto Deploy template](https://docs.gitlab.com/ee/topics/autodevops/upgrading_auto_deploy_dependencies.html#early-adopters)

编辑 .gitlab-ci.yml, 如下

```yaml
#https://docs.gitlab.com/ee/topics/autodevops/upgrading_auto_deploy_dependencies.html#verify-dependency-versions

include:
  - template: Auto-DevOps.gitlab-ci.yml #https://gitlab.com/gitlab-org/gitlab/-/blob/master/lib/gitlab/ci/templates/Jobs/Deploy.gitlab-ci.yml
  
  
.auto-deploy:
  image: "registry.gitlab.com/gitlab-org/cluster-integration/auto-deploy-image:v2.6.0"
```

> 此处定义的镜像，将会覆盖 Auto-DevOps的镜像。类似继承
>
> ```python
> class Base:
>     def abc():
>         print('git devops')
>     
> class Repo(Base):
>     def abc():
>         print('GitOps')
> ```
>
> 最终以仓库定义的为准



## [使用自定义的chart](https://docs.gitlab.com/ee/topics/autodevops/customize.html#custom-helm-chart)

- **Bundled chart** - If your project has a `./chart` directory with a `Chart.yaml` file in it, Auto DevOps detects the chart and uses it instead of the [default chart](https://gitlab.com/gitlab-org/cluster-integration/auto-deploy-image/-/tree/master/assets/auto-deploy-app), enabling you to control exactly how your application is deployed.

下载默认chart, 上传到仓库中的chart目录即可

```bash
git clone https://gitlab.com/gitlab-org/cluster-integration/auto-deploy-image.git
mv auto-deploy-image/assets/auto-deploy-app/ chart
 
# 确保charts目录中有Chart.yml文件
$ ls chart/
CONTRIBUTING.md  Chart.yaml  LICENSE  README.md  templates/  test/  values.yaml

$ rm chart/values.yaml -f
```

## 部署令牌

设置 - 仓库  - 部署令牌 ，拉镜像的默认账户。需要清理。在gitlab devops默认启动时会自动创建。

https://docs.gitlab.com/ee/topics/autodevops/stages.html#gitlab-deploy-tokens

If the GitLab Deploy Token can’t be found, `CI_REGISTRY_PASSWORD` is used.



## gitlab.yml reference

### allow_failure

Use `allow_failure` when you want to let a job fail without impacting the rest of the CI suite. The default value is `false`, except for [manual](https://docs.gitlab.com/ee/ci/yaml/#whenmanual) jobs that use the `when: manual` syntax.

In jobs that use [`rules:`](https://docs.gitlab.com/ee/ci/yaml/#rules), all jobs default to `allow_failure: false`, *including* `when: manual` jobs.

When `allow_failure` is set to `true` and the job fails, the job shows an orange warning in the UI. However, the logical flow of the pipeline considers the job a success/passed, and is not blocked.

Assuming all other jobs are successful, the job’s stage and its pipeline show the same orange warning. However, the associated commit is marked as “passed”, without warnings.

In the following example, `job1` and `job2` run in parallel. If `job1` fails, it doesn’t stop the next stage from running, because it’s marked with `allow_failure: true`:

```yaml
job1:
  stage: test
  script:
    - execute_script_that_will_fail
  allow_failure: true

job2:
  stage: test
  script:
    - execute_script_that_will_succeed

job3:
  stage: deploy
  script:
    - deploy_to_staging
```

### `allow_failure:exit_codes`

什么退出状态码不认为失败

```yaml
test_job_1:
  script:
    - echo "Run a script that results in exit code 1. This job fails."
    - exit 1
  allow_failure:
    exit_codes: 137

test_job_2:
  script:
    - echo "Run a script that results in exit code 137. This job is allowed to fail."
    - exit 137
  allow_failure:
    exit_codes:
      - 137
      - 255
```

### when

The valid values of `when` are:

1. `on_success` (default) - Execute job only when all jobs in earlier(初期的) stages succeed, or are considered successful because they have `allow_failure: true`.
2. `on_failure` - Execute job only when at least one job in an earlier stage fails.
3. `always` - Execute job regardless of the status of jobs in earlier stages.
4. `manual` - Execute job [manually](https://docs.gitlab.com/ee/ci/yaml/#whenmanual).
5. `delayed` - [Delay the execution of a job](https://docs.gitlab.com/ee/ci/yaml/#whendelayed) for a specified duration. Added in GitLab 11.14.
6. `never`:
   - With job [`rules`](https://docs.gitlab.com/ee/ci/yaml/#rules), don’t execute job.
   - With [`workflow:rules`](https://docs.gitlab.com/ee/ci/yaml/#workflow), don’t run pipeline.

In the following example, the script:

1. Executes `cleanup_build_job` only when `build_job` fails.
2. Always executes `cleanup_job` as the last step in pipeline regardless of success or failure.
3. Executes `deploy_job` when you run it manually in the GitLab UI.

```yaml
stages:
  - build
  - cleanup_build
  - test
  - deploy
  - cleanup

build_job:
  stage: build
  script:
    - make build

cleanup_build_job:
  stage: cleanup_build
  script:
    - cleanup build when failed
  when: on_failure

test_job:
  stage: test
  script:
    - make test

deploy_job:
  stage: deploy
  script:
    - make deploy
  when: manual

cleanup_job:
  stage: cleanup
  script:
    - cleanup after jobs
  when: always
```

#### `when:manual`

不会自动执行，而是显式地由用户启动。比如：部署到生产环境。

特性：

1. 默认manual的job的allow_failure为true.
2. 如果给manual的job添加allow_failure为false时，就会在job定义的阶段处阻塞。
3. rules中使用manual，allow_failure默认为false.
4. 

### rules

在pipeline中用来包含或排除job. 评估rules时，会有序的评估，一旦匹配到一个规则，要么是从pipeline排除这个job, 或者要么是包含这个job.

`rules` replaces [`only/except`](https://docs.gitlab.com/ee/ci/yaml/#only--except) and they can’t be used together in the same job. If you configure one job to use both keywords, the GitLab returns a `key may not be used with rules` error.

`rules` accepts an array of rules defined with:

- `if`
- `changes`
- `exists`
- `allow_failure`
- `variables`
- `when`

You can combine multiple keywords together for [complex rules](https://docs.gitlab.com/ee/ci/jobs/job_control.html#complex-rules).

The job is added to the pipeline:

- If an `if`, `changes`, or `exists` rule matches and also has `when: on_success` (default), `when: delayed`, or `when: always`.
- If a rule is reached that is only `when: on_success`, `when: delayed`, or `when: always`.

The job is not added to the pipeline:

- If no rules match.
- If a rule matches and has `when: never`

#### if

- If an `if` statement is true, add the job to the pipeline.
- If an `if` statement is true, but it’s combined with `when: never`, do not add the job to the pipeline.
- If no `if` statements are true, do not add the job to the pipeline.

`if:` clauses are evaluated based on the values of [predefined CI/CD variables](https://docs.gitlab.com/ee/ci/variables/predefined_variables.html) or [custom CI/CD variables](https://docs.gitlab.com/ee/ci/variables/index.html#custom-cicd-variables).



检查  `$CI_PIPELINE_SOURCE` 变量的值. https://docs.gitlab.com/ee/ci/jobs/job_control.html#common-if-clauses-for-rules

| Value                         | Description                                                  |
| :---------------------------- | :----------------------------------------------------------- |
| `api`                         | For pipelines triggered by the [pipelines API](https://docs.gitlab.com/ee/api/pipelines.html#create-a-new-pipeline). |
| `chat`                        | For pipelines created by using a [GitLab ChatOps](https://docs.gitlab.com/ee/ci/chatops/index.html) command. |
| `external`                    | When you use CI services other than GitLab.                  |
| `external_pull_request_event` | When an external pull request on GitHub is created or updated. See [Pipelines for external pull requests](https://docs.gitlab.com/ee/ci/ci_cd_for_external_repos/index.html#pipelines-for-external-pull-requests). |
| `merge_request_event`         | For pipelines created when a merge request is created or updated. Required to enable [merge request pipelines](https://docs.gitlab.com/ee/ci/pipelines/merge_request_pipelines.html), [merged results pipelines](https://docs.gitlab.com/ee/ci/pipelines/pipelines_for_merged_results.html), and [merge trains](https://docs.gitlab.com/ee/ci/pipelines/merge_trains.html). |
| `parent_pipeline`             | For pipelines triggered by a [parent/child pipeline](https://docs.gitlab.com/ee/ci/pipelines/parent_child_pipelines.html) with `rules`. Use this pipeline source in the child pipeline configuration so that it can be triggered by the parent pipeline. |
| `pipeline`                    | For [multi-project pipelines](https://docs.gitlab.com/ee/ci/pipelines/multi_project_pipelines.html) created by [using the API with `CI_JOB_TOKEN`](https://docs.gitlab.com/ee/ci/pipelines/multi_project_pipelines.html#create-multi-project-pipelines-by-using-the-api), or the [`trigger`](https://docs.gitlab.com/ee/ci/yaml/index.html#trigger) keyword. |
| `push`                        | For pipelines triggered by a `git push` event, including for branches and tags. |
| `schedule`                    | For [scheduled pipelines](https://docs.gitlab.com/ee/ci/pipelines/schedules.html). |
| `trigger`                     | For pipelines created by using a [trigger token](https://docs.gitlab.com/ee/ci/triggers/index.html#trigger-token). |
| `web`                         | For pipelines created by using **Run pipeline** button in the GitLab UI, from the project’s **CI/CD > Pipelines** section. |
| `webide`                      | For pipelines created by using the [WebIDE](https://docs.gitlab.com/ee/user/project/web_ide/index.html). |

```yaml
job:
  script: echo "Hello, Rules!"
  rules:
    - if: '$CI_PIPELINE_SOURCE == "schedule"'
      when: manual
      allow_failure: true
    - if: '$CI_PIPELINE_SOURCE == "push"'
```



`if` clauses:

- `if: $CI_COMMIT_TAG`: If changes are pushed for a tag.
- `if: $CI_COMMIT_BRANCH`: If changes are pushed to any branch.
- `if: '$CI_COMMIT_BRANCH == "main"'`: If changes are pushed to `main`.
- `if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'`: If changes are pushed to the default branch. Use when you want to have the same configuration in multiple projects with different default branches.
- `if: '$CI_COMMIT_BRANCH =~ /regex-expression/'`: If the commit branch matches a regular expression.
- `if: '$CUSTOM_VARIABLE !~ /regex-expression/'`: If the [custom variable](https://docs.gitlab.com/ee/ci/variables/index.html#custom-cicd-variables) `CUSTOM_VARIABLE` does **not** match a regular expression.
- `if: '$CUSTOM_VARIABLE == "value1"'`: If the custom variable `CUSTOM_VARIABLE` is exactly `value1`.







CI/CD variable expressions `rules:if`

```yaml
job1:
  variables:
    VAR1: "variable1"
  script:
    - echo "Test variable comparison
  rules:
    - if: $VAR1 == "variable1"
```

比较：前后不重要，字符使用单双引号

- `if: $VARIABLE == "some value"`
- `if: $VARIABLE != "some value"`
- `if: "some value" == $VARIABLE`

比较变量:

- `if: $VARIABLE_1 == $VARIABLE_2`
- `if: $VARIABLE_1 != $VARIABLE_2`

变量是否定义

- `if: $VARIABLE == null`
- `if: $VARIABLE != null`

变量是否为空串

- `if: $VARIABLE == ""`
- `if: $VARIABLE != ""`

变量是否存在

- `if: $VARIABLE`

正则

- `$VARIABLE =~ /^content.*/`       # 匹配
- `$VARIABLE_1 !~ /^content.*/` # 不匹配

组合

- `$VARIABLE1 =~ /^content.*/ && $VARIABLE2 == "something"`
- `$VARIABLE1 =~ /^content.*/ && $VARIABLE2 =~ /thing$/ && $VARIABLE3`
- `$VARIABLE1 =~ /^content.*/ || $VARIABLE2 =~ /thing$/ && $VARIABLE3`

括号区分优先级

- `($VARIABLE1 =~ /^content.*/ || $VARIABLE2) && ($VARIABLE3 =~ /thing$/ || $VARIABLE4)`
- `($VARIABLE1 =~ /^content.*/ || $VARIABLE2 =~ /thing$/) && $VARIABLE3`
- `$CI_COMMIT_BRANCH == "my-branch" || (($VARIABLE1 == "thing" || $VARIABLE2 == "thing") && $VARIABLE3)`

#### changes

Use `rules:changes` to specify when to add a job to a pipeline by checking for changes to specific files.

```yaml
docker build:
  variables:
    DOCKERFILES_DIR: 'path/to/files/'
  script: docker build -t my-image:$CI_COMMIT_REF_SLUG .
  rules:
    - changes:
        - $DOCKERFILES_DIR/*
```

You can use the `$` character for both variables and paths. For example, if the `$DOCKERFILES_DIR` variable exists, its value is used. If it does not exist, the `$` is interpreted as being part of a path

```yaml
docker build:
  script: docker build -t my-image:$CI_COMMIT_REF_SLUG .
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
      changes:
        - Dockerfile
      when: manual
      allow_failure: true
```

- If the pipeline is a merge request pipeline, check `Dockerfile` for changes.
- If `Dockerfile` has changed, add the job to the pipeline as a manual job, and the pipeline continues running even if the job is not triggered (`allow_failure: true`).
- If `Dockerfile` has not changed, do not add job to any pipeline (same as `when: never`).

#### exists

文件存在在仓库中，才运行job

列表，与项目目录相关的文件，不能引用外部的链接。可以使用glob模式通配

文件超过10000个时，总是匹配

```yaml
job:
  script: docker build -t my-image:$CI_COMMIT_REF_SLUG .
  rules:
    - exists:
        - Dockerfile
```

#### allow_failure

`allow_failure: true` 对没有停止的管道允许失败。默认是`false`, 即不定义allow_failure时，就是false.

You can also use `allow_failure: true` with a manual job. The pipeline continues running without waiting for the result of the manual job.  不等手工job的结果，就可以继续向后运行。

`allow_failure: false` combined with `when: manual` in rules causes the pipeline to wait for the manual job to run before continuing.

```yaml
job:
  script: echo "Hello, Rules!"
  rules:
    - if: '$CI_MERGE_REQUEST_TARGET_BRANCH_NAME == $CI_DEFAULT_BRANCH'
      when: manual
      allow_failure: true
```

- The rule-level `rules:allow_failure` overrides the job-level [`allow_failure`](https://docs.gitlab.com/ee/ci/yaml/#allow_failure), and only applies when the specific rule triggers the job.

#### variables

Use [`variables`](https://docs.gitlab.com/ee/ci/yaml/#variables) in `rules:` to define variables for specific conditions.

```yaml
job:
  variables:
    DEPLOY_VARIABLE: "default-deploy"
  rules:
    - if: $CI_COMMIT_REF_NAME == $CI_DEFAULT_BRANCH
      variables:                              # Override DEPLOY_VARIABLE defined
        DEPLOY_VARIABLE: "deploy-production"  # at the job level.
    - if: $CI_COMMIT_REF_NAME =~ /feature/
      variables:
        IS_A_FEATURE: "true"                  # Define a new variable.
  script:
    - echo "Run script with $DEPLOY_VARIABLE as an argument"
    - echo "Run another script if $IS_A_FEATURE exists"
```



#### potential to cause duplicate pipelines

```yaml
job:
  script: echo "This job creates double pipelines!"
  rules:
    - if: '$CUSTOM_VARIABLE == "false"'
      when: never
    - when: always
```

This job does not run when `$CUSTOM_VARIABLE` is false, but it *does* run in **all** other pipelines, including **both** push (branch) and merge request pipelines. With this configuration, every push to an open merge request’s source branch causes duplicated pipelines. 推送到打开merge request的源分支上，会造成merge request更新。所以会触发2次管道。

To avoid duplicate pipelines, you can:

- Use [`workflow`](https://docs.gitlab.com/ee/ci/yaml/index.html#workflow) to specify which types of pipelines can run.
- Rewrite the rules to run the job only in very specific cases, and avoid a final `when:` rule:

```yaml
job:
  script: echo "This job does NOT create double pipelines!"
  rules:
    - if: '$CUSTOM_VARIABLE == "true" && $CI_PIPELINE_SOURCE == "merge_request_event"'
```

You can also avoid duplicate pipelines by changing the job rules to avoid either push (branch) pipelines or merge request pipelines. However, if you use a `- when: always` rule without `workflow: rules`, GitLab still displays a [pipeline warning](https://docs.gitlab.com/ee/ci/troubleshooting.html#pipeline-warnings).



### 变量

Variables are meant for non-sensitive(敏感) project configuration, for example:

If you define a variable at the top level of the `gitlab-ci.yml` file, it is global, meaning it applies to all jobs. If you define a variable in a job, it’s available to that job only.

If a variable of the same name is defined globally and for a specific job, the [job-specific variable overrides the global variable](https://docs.gitlab.com/ee/ci/variables/index.html#cicd-variable-precedence).

All YAML-defined variables are also set to any linked [Docker service containers](https://docs.gitlab.com/ee/ci/services/index.html).

You can use [YAML anchors for variables](https://docs.gitlab.com/ee/ci/yaml/index.html#yaml-anchors-for-variables).

#### 全局和局部变量

```yaml
variables:
  DEPLOY_SITE: "https://example.com/"

deploy_job:
  stage: deploy
  script:
    - deploy-script --url $DEPLOY_SITE --path "/"

deploy_review_job:
  stage: deploy
  variables:
    REVIEW_PATH: "/review"
  script:
    - deploy-review-script --url $DEPLOY_SITE --path $REVIEW_PATH
```

#### 变量在服务中可用

```yaml
# The following variables are automatically passed down to the Postgres container
# as well as the Ruby container and available within each.
variables:
  HTTPS_PROXY: "https://10.1.1.1:8090"
  HTTP_PROXY: "https://10.1.1.1:8090"
  POSTGRES_DB: "my_custom_db"
  POSTGRES_USER: "postgres"
  POSTGRES_PASSWORD: "example"
  PGDATA: "/var/lib/postgresql/data"
  POSTGRES_INITDB_ARGS: "--encoding=UTF8 --data-checksums"

abc:
    services:
      - name: postgres:11.7
        alias: db
        entrypoint: ["docker-entrypoint.sh"]
        command: ["postgres"]

    image:
      name: ruby:2.6
      entrypoint: ["/bin/bash"]

    before_script:
      - bundle install

    test:
      script:
        - bundle exec rake spec
```



###  隐藏

 you can start its name with a dot (`.`) and it is not processed by GitLab CI/CD. In the following example, `.hidden_job` is ignored:

```yaml
.hidden_job:
  script:
    - run test
```

### 继承

重用配置，使用带&符的 隐藏模板。&不能跨文件使用，只能在定义的文件中使用。 

跨文件引用配置： [`!reference` tags](https://docs.gitlab.com/ee/ci/yaml/index.html#reference-tags) or the [`extends` keyword](https://docs.gitlab.com/ee/ci/yaml/index.html#extends).

#### 单文件继承

##### 继承所有

使用`&`和`<<`,  It creates two jobs, `test1` and `test2`, that inherit the `.job_template` configuration, each with their own custom `script` defined:

```yaml
.job_template: &job_configuration  # Hidden yaml configuration that defines an anchor named 'job_configuration'
  image: ruby:2.6
  services:
    - postgres
    - redis

test1:
  <<: *job_configuration           # Merge the contents of the 'job_configuration' alias
  script:
    - test1 project

test2:
  <<: *job_configuration           # Merge the contents of the 'job_configuration' alias
  script:
    - test2 project
```

> &给隐藏模板设定一个锚写名。<< *锚定名称，将会合并给定的hash到当前。
>
> 结果如下:
>
> ```yaml
> .job_template:
>   image: ruby:2.6
>   services:
>     - postgres
>     - redis
> 
> test1:
>   image: ruby:2.6
>   services:
>     - postgres
>     - redis
>   script:
>     - test1 project
> 
> test2:
>   image: ruby:2.6
>   services:
>     - postgres
>     - redis
>   script:
>     - test2 project
> ```

##### 继承1个key

`test:postgres` and `test:mysql` share the `script` defined in `.job_template`，but use different `services`, defined in `.postgres_services` and `.mysql_services`:

```yaml
.job_template: &job_configuration
  script:
    - test project
  tags:
    - dev

.postgres_services:
  services: &postgres_configuration
    - postgres
    - ruby

.mysql_services:
  services: &mysql_configuration
    - mysql
    - ruby

test:postgres:
  <<: *job_configuration
  services: *postgres_configuration
  tags:
    - postgres

test:mysql:
  <<: *job_configuration
  services: *mysql_configuration
```

> .开始不会执行，其中的key, 优先级低于被引用的job。

#### 跨文件继承

要重用其他文件中的配置，事先需要`include` 其他文件，再引用。

##### `!reference tags` and `include`

`setup.yml`

```yaml
.setup:
  script:
    - echo creating environment
```

`.gitlab-ci.yml`:

```yaml
include:
  - local: setup.yml # 先include

.teardown:
  after_script:
    - echo deleting environment

test:
  script:
    - !reference [.setup, script] # 引用上面脚本
    - echo running my own command
  after_script:
    - !reference [.teardown, after_script] # 同文件也可以引用
```

In the following example, `test-vars-1` reuses all the variables in `.vars`, while `test-vars-2` selects a specific variable and reuses it as a new `MY_VAR` variable.

```yaml
.vars:
  variables:
    URL: "http://my-url.internal"
    IMPORTANT_VAR: "the details"

test-vars-1:
  variables: !reference [.vars, variables]
  script:
    - printenv

test-vars-2:
  variables:
    MY_VAR: !reference [.vars, variables, IMPORTANT_VAR]
  script:
    - printenv
```

不能重用一个包含了`!reference`tag的job, `!reference`仅仅支持一级嵌套。

### 部分继承

In the following example:

- `rubocop`:
  - inherits: Nothing.
- `rspec`:
  - inherits: the default `image` and the `WEBHOOK_URL` variable.
  - does **not** inherit: the default `before_script` and the `DOMAIN` variable.
- `capybara`:
  - inherits: the default `before_script` and `image`.
  - does **not** inherit: the `DOMAIN` and `WEBHOOK_URL` variables.
- `karma`:
  - inherits: the default `image` and `before_script`, and the `DOMAIN` variable.
  - does **not** inherit: `WEBHOOK_URL` variable.

```yaml
default:
  image: 'ruby:2.4'
  before_script:
    - echo Hello World

variables:
  DOMAIN: example.com
  WEBHOOK_URL: https://my-webhook.example.com

rubocop:
  inherit:
    default: false
    variables: false
  script: bundle exec rubocop

rspec:
  inherit:
    default: [image]
    variables: [WEBHOOK_URL]
  script: bundle exec rspec

capybara:
  inherit:
    variables: false
  script: bundle exec capybara

karma:
  inherit:
    default: true
    variables: [DOMAIN]
  script: karma
```



### skip pipeline

推送commit, 不触发一个pipeline, 添加add `[ci skip]` or `[skip ci]`, using any 大小写(capitalization), to your commit message.

Alternatively, if you are using Git 2.10 or later, use the `ci.skip` [Git push option](https://docs.gitlab.com/ee/user/project/push_options.html#push-options-for-gitlab-cicd). The `ci.skip` push option does not skip merge request pipelines.

### Processing Git pushes

GitLab creates at most four branch and tag pipelines when pushing multiple changes in a single `git push` invocation.

This limitation does not affect any of the updated merge request pipelines. All updated merge requests have a pipeline created when using [pipelines for merge requests](https://docs.gitlab.com/ee/ci/pipelines/merge_request_pipelines.html).

### 全局定义`image`, `services`, `cache`, `before_script`, `after_script`

Use [`default:`](https://docs.gitlab.com/ee/ci/yaml/index.html#custom-default-keyword-values) . For example:

```yaml
default:
  image: ruby:3.0
  services:
    - docker:dind
  cache:
    paths: [vendor/]
  before_script:
    - bundle config set path vendor/bundle
    - bundle install
  after_script:
    - rm -rf tmp/
```

### 检验gitlib-ci.yml语法

To access the CI Lint tool, navigate to **CI/CD > Pipelines** or **CI/CD > Jobs** in your project and click **CI lint**

paste a complete CI configuration (`.gitlab-ci.yml` for example) into the text box and click **Validate**:



## 优化镜像，使用阿里镜像

```bash
docker pull   registry.gitlab.com/gitlab-org/cluster-integration/auto-build-image/master:stable
docker tag registry.gitlab.com/gitlab-org/cluster-integration/auto-build-image/master:stable registry.cn-hangzhou.aliyuncs.com/grasp_base_images/auto-build-image:stable
docker push registry.cn-hangzhou.aliyuncs.com/grasp_base_images/auto-build-image:stable

docker pull registry.gitlab.com/gitlab-org/cluster-integration/auto-deploy-image:v2.6.0
docker tag registry.gitlab.com/gitlab-org/cluster-integration/auto-deploy-image:v2.6.0 registry.cn-hangzhou.aliyuncs.com/grasp_base_images/auto-deploy-image:v2.6.0
docker push registry.cn-hangzhou.aliyuncs.com/grasp_base_images/auto-deploy-image:v2.6.0

docker pull codeclimate/codeclimate:0.85.5
docker tag codeclimate/codeclimate:0.85.5 registry.cn-hangzhou.aliyuncs.com/grasp_base_images/codeclimate:0.85.5
docker push registry.cn-hangzhou.aliyuncs.com/grasp_base_images/codeclimate:0.85.5

docker pull registry.gitlab.com/gitlab-org/security-products/codequality:12-5-stable
docker tag registry.gitlab.com/gitlab-org/security-products/codequality:12-5-stable registry.cn-hangzhou.aliyuncs.com/grasp_base_images/codequality:12-5-stable
docker push registry.cn-hangzhou.aliyuncs.com/grasp_base_images/codequality:12-5-stable



docker tag docker:stable registry.cn-hangzhou.aliyuncs.com/grasp_base_images/docker:stable
docker push registry.cn-hangzhou.aliyuncs.com/grasp_base_images/docker:stable

docker tag docker:stable-dind registry.cn-hangzhou.aliyuncs.com/grasp_base_images/docker:stable-dind
docker push registry.cn-hangzhou.aliyuncs.com/grasp_base_images/docker:stable-dind

docker tag sitespeedio/sitespeed.io:6.3.1 registry.cn-hangzhou.aliyuncs.com/grasp_base_images/sitespeed.io:6.3.1
docker push registry.cn-hangzhou.aliyuncs.com/grasp_base_images/sitespeed.io:6.3.1


docker pull sitespeedio/sitespeed.io:14.1.0
docker tag sitespeedio/sitespeed.io:14.1.0 registry.cn-hangzhou.aliyuncs.com/grasp_base_images/sitespeed.io:14.1.0
docker push registry.cn-hangzhou.aliyuncs.com/grasp_base_images/sitespeed.io:14.1.0
```

### 重做codequality

```dockerfile
[root@iZbp194wn395fdozcybt6gZ ~]# cat Dockerfile 
FROM registry.cn-hangzhou.aliyuncs.com/grasp_base_images/codequality:12-5-stable

RUN sed -i 's@codeclimate/codeclimate:@registry.cn-hangzhou.aliyuncs.com/grasp_base_images/codeclimate:@g' run.sh
```

测试

```bash
# docker build -t test:v123 .

# 查看是否更新
# docker run --rm -it --entrypoint=/bin/sh test:v123  -c "cat run.sh"

[root@iZbp194wn395fdozcybt6gZ ~]# docker tag test:v123 registry.cn-hangzhou.aliyuncs.com/grasp_base_images/codequality:12-5-stable
[root@iZbp194wn395fdozcybt6gZ ~]# docker push registry.cn-hangzhou.aliyuncs.com/grasp_base_images/codequality:12-5-stable


[root@iZbp194wn395fdozcybt6gZ ~]# docker tag codeclimate/codeclimate:0.85.24 registry.cn-hangzhou.aliyuncs.com/grasp_base_images/codeclimate:0.85.24
[root@iZbp194wn395fdozcybt6gZ ~]# docker push registry.cn-hangzhou.aliyuncs.com/grasp_base_images/codeclimate:0.85.24
```

> 更新版本，通过gitlab-ci.yml文件传递环境变量来更新

### 重做deploy镜像

registry.cn-hangzhou.aliyuncs.com/grasp_base_images/auto-deploy-image:v2.6.0

此镜像由于本身，只需要chart部署，但是部署前需要添加一个仓库，导致非常慢。所以去年此步

```bash
root@51a60b42fd47:/data/songliangcheng/auto-deploy-image# cat Dockerfile 
FROM registry.cn-hangzhou.aliyuncs.com/grasp_base_images/auto-deploy-image:v2.6.0
RUN  sed -i  's@helm repo add stable https://charts.helm.sh/stable@#&@g' /usr/local/bin/auto-deploy 
```

```bash
docker build -t registry.cn-hangzhou.aliyuncs.com/grasp_base_images/auto-deploy-image:v2.6.0 .
docker push registry.cn-hangzhou.aliyuncs.com/grasp_base_images/auto-deploy-image:v2.6.0 
```





GitLab版本: v12.5.2-ee,  对应的模板

https://gitlab.com/gitlab-org/gitlab/-/blob/v12.5.2-ee/lib/gitlab/ci/templates/Jobs/Deploy.gitlab-ci.yml

> 注意：不同的版本，模板中继承模板方式不一样。

```yaml
# 原生gitlab.yml就是Auto-Devops.gitlab-ci.yml , 链接在: https://docs.gitlab.com/ee/topics/autodevops/upgrading_auto_deploy_dependencies.html#verify-dependency-versions

# gitlab-ci.yml 文档 https://docs.gitlab.com/ee/ci/yaml/index.html

include:
- template: Auto-DevOps.gitlab-ci.yml #https://gitlab.com/gitlab-org/gitlab/-/blob/master/lib/gitlab/ci/templates/Jobs/Deploy.gitlab-ci.yml

  
.auto-deploy:
  image: "registry.cn-hangzhou.aliyuncs.com/grasp_base_images/auto-deploy-image:v2.6.0"

build:
  stage: build
  image: 'registry.cn-hangzhou.aliyuncs.com/grasp_base_images/auto-build-image:stable'

code_quality:
  stage: test
  image: registry.cn-hangzhou.aliyuncs.com/grasp_base_images/docker:stable
  script:
    - |
      if ! docker info &>/dev/null; then
        if [ -z "$DOCKER_HOST" -a "$KUBERNETES_PORT" ]; then
          export DOCKER_HOST='tcp://localhost:2375'
        fi
      fi
    - docker run
        --env SOURCE_CODE="$PWD"
        --env CODECLIMATE_VERSION="${CODECLIMATE_VERSION}"
        --volume "$PWD":/code
        --volume /var/run/docker.sock:/var/run/docker.sock
        "registry.cn-hangzhou.aliyuncs.com/grasp_base_images/codequality:12-5-stable" /code


performance:
  stage: performance
  image: registry.cn-hangzhou.aliyuncs.com/grasp_base_images/docker:stable
  allow_failure: true
  variables:
    DOCKER_TLS_CERTDIR: ""
  services:
    - registry.cn-hangzhou.aliyuncs.com/grasp_base_images/docker:stable-dind
  script:
    - |
      if ! docker info &>/dev/null; then
        if [ -z "$DOCKER_HOST" -a "$KUBERNETES_PORT" ]; then
          export DOCKER_HOST='tcp://localhost:2375'
        fi
      fi
    - export CI_ENVIRONMENT_URL=$(cat environment_url.txt)
    - mkdir gitlab-exporter
    - wget -O gitlab-exporter/index.js https://gitlab.com/gitlab-org/gl-performance/raw/10-5/index.js
    - mkdir sitespeed-results
    - |
      if [ -f .gitlab-urls.txt ]
      then
        sed -i -e 's@^@'"$CI_ENVIRONMENT_URL"'@' .gitlab-urls.txt
        docker run --shm-size=1g --rm -v "$(pwd)":/sitespeed.io registry.cn-hangzhou.aliyuncs.com/grasp_base_images/sitespeed.io:6.3.1 --plugins.add ./gitlab-exporter --outputFolder sitespeed-results .gitlab-urls.txt
      else
        docker run --shm-size=1g --rm -v "$(pwd)":/sitespeed.io registry.cn-hangzhou.aliyuncs.com/grasp_base_images/sitespeed.io:6.3.1 --plugins.add ./gitlab-exporter --outputFolder sitespeed-results "$CI_ENVIRONMENT_URL"
      fi
    - mv sitespeed-results/data/performance.json performance.json


variables:
  POSTGRES_VERSION: "9.6.16"
  CODECLIMATE_VERSION: "v0.85.24" # 更新质量扫描的版本
```

## 代码扫描加速

https://docs.gitlab.com/ee/user/project/merge_requests/code_quality.html

To ensure your project’s code stays simple, readable, and easy to contribute to, you can use [GitLab CI/CD](https://docs.gitlab.com/ee/ci/index.html) to analyze your source code quality.

你写了一个新特性，使用代码质量报告分析，会提示你如何提高代码质量，并且给出性能评估

gitlab会把代码质量报告显示在merge request挂件区域。

可支持的语言: https://docs.codeclimate.com/docs/supported-languages-for-maintainability

**Languages:**

- Ruby
- Python
- PHP
- JavaScript
- Java
- TypeScript
- GoLang
- Swift
- Scala
- Kotlin
- C#

### 代码质量的差异意见

merge 请求的文件的改变，如果进行merge,可能会导致代码质量的下降，在这种情况下，merge request的差异意见 会紧挨着行显示一个指示。一个新代码质量的不雅行为。例如：

![diff view](https://docs.gitlab.com/ee/user/project/merge_requests/img/code_quality_mr_diff_report_v14_2.png)

> 严重：考虑简化这个复杂的表达式。

### 配置示例

代码报告位置：https://docs.gitlab.com/12.10/ee/user/project/merge_requests/code_quality.html#code-quality-reports

- pipeline详情页
- merge request挂件页，只要不是单纯的修改.gitlab-ci.yml文件，一般会显示当前和分支head比较后的报告。列出当merge之后将被解决或创建的缺点。

This example shows how to run Code Quality on your code by using GitLab CI/CD and Docker. It requires GitLab 11.11 or later, and GitLab Runner 11.5 or later. If you are using GitLab 11.4 or earlier, you can view the deprecated job definitions in the [documentation archive](https://docs.gitlab.com/12.10/ee/user/project/merge_requests/code_quality.html#previous-job-definitions).

- Using shared runners, the job should be configured For the [Docker-in-Docker workflow](https://docs.gitlab.com/ee/ci/docker/using_docker_build.html#use-the-docker-executor-with-the-docker-image-docker-in-docker).
- Using private runners, there is an [alternative configuration](https://docs.gitlab.com/ee/user/project/merge_requests/code_quality.html#set-up-a-private-runner-for-code-quality-without-docker-in-docker) recommended for running Code Quality analysis more efficiently.

### docker-in-docker 局限性

- **The `docker-compose` command**: This command is not available in this configuration by default. To use `docker-compose` in your job scripts, follow the `docker-compose` [installation instructions](https://docs.docker.com/compose/install/).
- **Cache**: Each job runs in a new environment. Concurrent(并发) jobs work fine, because every build gets its own instance of Docker engine and they don’t conflict with each other. However, jobs can be slower(慢) because there’s no caching of layers.
- **Storage drivers**: By default, earlier versions of Docker use the `vfs` storage driver, which copies the file system for each job. Docker 17.09 and later use `--storage-driver overlay2`, which is the recommended storage driver. See [Using the OverlayFS driver](https://docs.gitlab.com/ee/ci/docker/using_docker_build.html#use-the-overlayfs-driver) for details.
- **Root file system**: Because the `docker:19.03.12-dind` container and the runner container don’t share their root file system, you can use the job’s working directory as a mount point for child containers. For example, if you have files you want to share with a child container, you might create a subdirectory under `/builds/$CI_PROJECT_PATH` and use it as your mount point. For a more detailed explanation, view [issue #41227](https://gitlab.com/gitlab-org/gitlab-foss/-/issues/41227).

```yaml
variables:
  MOUNT_POINT: /builds/$CI_PROJECT_PATH/mnt
script:
  - mkdir -p "$MOUNT_POINT"
  - docker run -v "$MOUNT_POINT:/mnt" my-docker-image
```

### Set up a private runner for code quality without Docker-in-Docker

 You can use a configuration that may greatly **speed up**(使加速) job execution without requiring your runners to operate in privileged mode.

This alternative configuration uses socket binding to share the Runner’s Docker daemon with the job environment. Be aware that this configuration [has significant considerations](https://docs.gitlab.com/ee/ci/docker/using_docker_build.html#use-docker-socket-binding) to be consider, but may be preferable depending on your use case.

先在gitlab ci/cd中重置token, 再

helm创建的runner, 使用以下配置

```bash
pos@1227f652bef0:~/.cache/helm/repository$ cp -a gitlab-runner gitlab-runner-without-docker-in-docker
```

>  配置host_path, 详情参考： https://docs.gitlab.com/runner/executors/kubernetes.html#host-path-volumes

```diff
imagePullPolicy: IfNotPresent
replicas: 1
gitlabUrl: http://182.148.48.146:9292
runnerRegistrationToken: "ocoTP9iQ9xben9E-Zk_T"
terminationGracePeriodSeconds: 3600
concurrent: 10
checkInterval: 30
rbac:
  create: true
  rules: 
  - resources: ["pods", "secrets"]
    verbs: ["get", "list", "watch", "create", "patch", "delete"]
  - apiGroups: [""]
    resources: ["pods/exec"]
    verbs: ["create", "patch", "delete"]
  clusterWideAccess: false
  serviceAccountName: default
  podSecurityPolicy:
    enabled: false
    resourceNames:
    - gitlab-runner
metrics:
  enabled: true
runners:
  config: |
    [[runners]]
      clone_url = "http://182.148.48.146:9292"
      url = "http://182.148.48.146:9292"
      name = "cq-sans-dind"
      token = "ocoTP9iQ9xben9E-Zk_T"
      executor = "kubernetes"
      builds_dir = "/tmp/builds"
      [runners.docker]
          tls_verify = false
          image = "registry.cn-hangzhou.aliyuncs.com/grasp_base_images/docker:stable"
          privileged = false
          disable_entrypoint_overwrite = false
          oom_kill_disable = false
          disable_cache = false
          volumes = ["/cache", "/var/run/docker.sock:/var/run/docker.sock", "/tmp/builds:/tmp/builds"]
          shm_size = 0
      [runners.kubernetes]
        namespace = "{{.Release.Namespace}}"
        image = "ubuntu:16.04"
        [[runners.kubernetes.volumes.host_path]]
          name = "docker-on-host"
          mount_path = "/var/run/docker.sock"
          read_only = true
        [[runners.kubernetes.volumes.host_path]]
          name = "build-on-host"
          mount_path = "/tmp/builds" 
          read_only = false
        [[runners.kubernetes.volumes.host_path]]
          name = "cache-on-host"
          mount_path = "/cache" 
          read_only = false
  executor: kubernetes
  tags: "cq-sans-dind"
  name: "cq-sans-dind"
  privileged: true
  cache: {}
  builds: {}
  services: {}
  helpers: {}
  serviceAccountName: default
securityContext:
  runAsUser: 100
  fsGroup: 65533
resources: {}
affinity: {}
nodeSelector: {}
tolerations: []
hostAliases: []
podAnnotations: {}
podLabels: {}
secrets: []
configMaps: {}
```

```bash
pos@1227f652bef0:~/.cache/helm/repository$ helm install -n gitlab gitlab-runner-cq ./gitlab-runner-without-docker-in-docker/
NAME: gitlab-runner-cq
LAST DEPLOYED: Wed Aug 11 01:44:26 2021
NAMESPACE: gitlab
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Your GitLab Runner should now be registered against the GitLab instance reachable at:

Runner namespace "gitlab" was found in runners.config template.
pos@1227f652bef0:~/.cache/helm/repository$ kubectl get pod -n gitlab
NAME                                             READY   STATUS    RESTARTS   AGE
gitlab-runner-cq-gitlab-runner-bf584f59b-bjkqp   0/1     Running   0          8s
```

Apply two overrides to the `code_quality` job created by the template:

```
include:
  - template: Code-Quality.gitlab-ci.yml

code_quality:
  services:            # Shut off Docker-in-Docker
  tags:
    - cq-sans-dind     # Set this job to only run on our new specialized runner
```

The end result is that:

- Privileged mode is not used.
- Docker-in-Docker is not used.
- Docker images, including all CodeClimate images, are cached, and not re-fetched for subsequent jobs.

With this configuration, the run time for a second(第2次的) pipeline is much shorter. For example this [small change](https://gitlab.com/drew/test-code-quality-template/-/merge_requests/4/diffs?commit_id=1e705607aef7236c1b20bb6f637965f3f3e53a46) to an [open merge request](https://gitlab.com/drew/test-code-quality-template/-/merge_requests/4/pipelines) running Code Quality analysis ran significantly faster the second time:

![image-20210811113104020](http://myapp.img.mykernel.cn/image-20210811113104020.png)

This configuration is not possible on `gitlab.com` shared runners. Shared runners are configured with `privileged=true`, and they do not expose `docker.sock` into the job container. As a result, socket binding cannot be used to make `docker` available in the context of the job script.

[Docker-in-Docker](https://docs.gitlab.com/ee/ci/docker/using_docker_build.html#use-the-docker-executor-with-the-docker-image-docker-in-docker) was chosen as an operational decision by the runner team, instead of exposing `docker.sock`

### 最终质量扫描

```yaml
code_quality:
  tags:
  - cq-sans-dind
  stage: test
  image: registry.cn-hangzhou.aliyuncs.com/grasp_base_images/docker:stable
  allow_failure: true
  services:
  script:
    - |
      if ! docker info &>/dev/null; then
        if [ -z "$DOCKER_HOST" -a "$KUBERNETES_PORT" ]; then
          export DOCKER_HOST='tcp://localhost:2375'
        fi
      fi
    - docker run
        --env SOURCE_CODE="$PWD"
        --env CODECLIMATE_VERSION="${CODECLIMATE_VERSION}"
        --volume "$PWD":/code
        --volume /var/run/docker.sock:/var/run/docker.sock
        "registry.cn-hangzhou.aliyuncs.com/grasp_base_images/codequality:12-5-stable" /code

```



## gitlab ci.yaml

```yaml
# 原生gitlab.yml就是Auto-Devops.gitlab-ci.yml , 链接在: https://docs.gitlab.com/ee/topics/autodevops/upgrading_auto_deploy_dependencies.html#verify-dependency-versions

# gitlab-ci.yml 文档 https://docs.gitlab.com/ee/ci/yaml/index.html

include:
- template: Auto-DevOps.gitlab-ci.yml #https://gitlab.com/gitlab-org/gitlab/-/blob/master/lib/gitlab/ci/templates/Jobs/Deploy.gitlab-ci.yml

  
.auto-deploy:
  image: "registry.cn-hangzhou.aliyuncs.com/grasp_base_images/auto-deploy-image:v2.6.0"

build:
  stage: build
  image: 'registry.cn-hangzhou.aliyuncs.com/grasp_base_images/auto-build-image:stable'

code_quality:
  tags:
  - cq-sans-dind
  stage: test
  image: registry.cn-hangzhou.aliyuncs.com/grasp_base_images/docker:stable
  allow_failure: true
  services:
  script:
    - |
      if ! docker info &>/dev/null; then
        if [ -z "$DOCKER_HOST" -a "$KUBERNETES_PORT" ]; then
          export DOCKER_HOST='tcp://localhost:2375'
        fi
      fi
    - docker run
        --env SOURCE_CODE="$PWD"
        --env CODECLIMATE_VERSION="${CODECLIMATE_VERSION}"
        --volume "$PWD":/code
        --volume /var/run/docker.sock:/var/run/docker.sock
        "registry.cn-hangzhou.aliyuncs.com/grasp_base_images/codequality:12-5-stable" /code

  
  
performance:
  image: registry.cn-hangzhou.aliyuncs.com/grasp_base_images/docker:stable
  allow_failure: true
  variables:
    DOCKER_TLS_CERTDIR: ""
    SITESPEED_IMAGE: registry.cn-hangzhou.aliyuncs.com/grasp_base_images/sitespeed.io
    SITESPEED_VERSION: 14.1.0
    SITESPEED_OPTIONS: ''
  services:
    - registry.cn-hangzhou.aliyuncs.com/grasp_base_images/docker:stable-dind
  script:
    - |
      if ! docker info &>/dev/null; then
        if [ -z "$DOCKER_HOST" -a "$KUBERNETES_PORT" ]; then
          export DOCKER_HOST='tcp://localhost:2375'
        fi
      fi
    - export CI_ENVIRONMENT_URL=$(cat environment_url.txt)
    - mkdir gitlab-exporter
    # Busybox wget does not support proxied HTTPS, get the real thing.
    # See https://gitlab.com/gitlab-org/gitlab/-/issues/287611.
    - (env | grep -i _proxy= 2>&1 >/dev/null) && apk --no-cache add wget
    - wget -O gitlab-exporter/index.js https://gitlab.com/gitlab-org/gl-performance/raw/1.1.0/index.js
    - mkdir sitespeed-results
    - |
      function propagate_env_vars() {
        CURRENT_ENV=$(printenv)

        for VAR_NAME; do
          echo $CURRENT_ENV | grep "${VAR_NAME}=" > /dev/null && echo "--env $VAR_NAME "
        done
      }
    - |
      if [ -f .gitlab-urls.txt ]
      then
        sed -i -e 's@^@'"$CI_ENVIRONMENT_URL"'@' .gitlab-urls.txt
        docker run \
          $(propagate_env_vars \
            auto_proxy \
            https_proxy \
            http_proxy \
            no_proxy \
            AUTO_PROXY \
            HTTPS_PROXY \
            HTTP_PROXY \
            NO_PROXY \
          ) \
          --shm-size=1g --rm -v "$(pwd)":/sitespeed.io $SITESPEED_IMAGE:$SITESPEED_VERSION --plugins.add ./gitlab-exporter --cpu --outputFolder sitespeed-results .gitlab-urls.txt $SITESPEED_OPTIONS
      else
        docker run \
          $(propagate_env_vars \
            auto_proxy \
            https_proxy \
            http_proxy \
            no_proxy \
            AUTO_PROXY \
            HTTPS_PROXY \
            HTTP_PROXY \
            NO_PROXY \
          ) \
          --shm-size=1g --rm -v "$(pwd)":/sitespeed.io $SITESPEED_IMAGE:$SITESPEED_VERSION --plugins.add ./gitlab-exporter --cpu --outputFolder sitespeed-results "$CI_ENVIRONMENT_URL" $SITESPEED_OPTIONS
      fi
    - mv sitespeed-results/data/performance.json browser-performance.json
  artifacts:
    paths:
      - sitespeed-results/
      - browser-performance.json



variables:
  POSTGRES_VERSION: "9.6.16"
  CODECLIMATE_VERSION: "v0.85.24" # 更新质量扫描的版本

```

## 示例，merge

此处会显示代码质量

![image-20210813115711480](http://myapp.img.mykernel.cn/image-20210813115711480.png)

## 合并之后

生成新的流水线

