# digdag-operator-ecs_task
[![Jitpack](https://jitpack.io/v/pro.civitaspo/digdag-operator-ecs_task.svg)](https://jitpack.io/#pro.civitaspo/digdag-operator-ecs_task) [![CircleCI](https://circleci.com/gh/civitaspo/digdag-operator-ecs_task.svg?style=shield)](https://circleci.com/gh/civitaspo/digdag-operator-ecs_task) [![Digdag](https://img.shields.io/badge/digdag-v0.9.31-brightgreen.svg)](https://github.com/treasure-data/digdag/releases/tag/v0.9.31)

digdag plugin for AWS ECS Task.

# Overview

- Plugin type: operator

# Usage

```yaml
_export:
  plugin:
    repositories:
      - https://jitpack.io
    dependencies:
      - pro.civitaspo:digdag-operator-ecs_task:0.0.3
  ecs_task:
    auth_method: profile

+step0:
  sh>: echo '{"store_params":{"civi":"taspo"}}' | aws s3 cp - ${output}

+step1:
  ecs_task.run>:
  def:
    network_mode: Host
    container_definitions:
      - name: uploader
        image: amazonlinux:2
        command: [yum, install, '-y', awscli]
        essential: true
        memory: 500
        cpu: 10
    family: hello_world
  cluster: ${cluster}
  count: 1
  result_s3_uri: ${output}

+step2:
  echo>: ${civi}

```

# Configuration

## Remarks

- type `DurationParam` is strings matched `\s*(?:(?<days>\d+)\s*d)?\s*(?:(?<hours>\d+)\s*h)?\s*(?:(?<minutes>\d+)\s*m)?\s*(?:(?<seconds>\d+)\s*s)?\s*`.
  - The strings is used as `java.time.Duration`.

## Common Configuration

### System Options

Define the below options on properties (which is indicated by `-c`, `--config`).

- **ecs_task.allow_auth_method_env**: Indicates whether users can use **auth_method** `"env"` (boolean, default: `false`)
- **ecs_task.allow_auth_method_instance**: Indicates whether users can use **auth_method** `"instance"` (boolean, default: `false`)
- **ecs_task.allow_auth_method_profile**: Indicates whether users can use **auth_method** `"profile"` (boolean, default: `false`)
- **ecs_task.allow_auth_method_properties**: Indicates whether users can use **auth_method** `"properties"` (boolean, default: `false`)
- **ecs_task.assume_role_timeout_duration**: Maximum duration which server administer allows when users assume **role_arn**. (`DurationParam`, default: `1h`)

### Secrets

- **ecs_task.access_key_id**: The AWS Access Key ID (optional)
- **ecs_task.secret_access_key**: The AWS Secret Access Key (optional)
- **ecs_task.session_token**: The AWS session token. This is used only **auth_method** is `"session"` (optional)
- **ecs_task.role_arn**: The AWS Role to assume. (optional)
- **ecs_task.role_session_name**: The AWS Role Session Name when assuming the role. (default: `digdag-ecs_task-${session_uuid}`)
- **ecs_task.http_proxy.host**: proxy host (required if **use_http_proxy** is `true`)
- **ecs_task.http_proxy.port** proxy port (optional)
- **ecs_task.http_proxy.scheme** `"https"` or `"http"` (default: `"https"`)
- **ecs_task.http_proxy.user** proxy user (optional)
- **ecs_task.http_proxy.password**: http proxy password (optional)

### Options

- **auth_method**: name of mechanism to authenticate requests (`"basic"`, `"env"`, `"instance"`, `"profile"`, `"properties"`, `"anonymous"`, or `"session"`. default: `"basic"`)
  - `"basic"`: uses access_key_id and secret_access_key to authenticate.
  - `"env"`: uses AWS_ACCESS_KEY_ID (or AWS_ACCESS_KEY) and AWS_SECRET_KEY (or AWS_SECRET_ACCESS_KEY) environment variables.
  - `"instance"`: uses EC2 instance profile.
  - `"profile"`: uses credentials written in a file. Format of the file is as following, where `[...]` is a name of profile.
    - **profile_file**: path to a profiles file. (string, default: given by `AWS_CREDENTIAL_PROFILES_FILE` environment varialbe, or ~/.aws/credentials).
    - **profile_name**: name of a profile. (string, default: `"default"`)
  - `"properties"`: uses aws.accessKeyId and aws.secretKey Java system properties.
  - `"anonymous"`: uses anonymous access. This auth method can access only public files.
  - `"session"`: uses temporary-generated access_key_id, secret_access_key and session_token.
- **use_http_proxy**: Indicate whether using when accessing AWS via http proxy. (boolean, default: `false`)
- **region**: The AWS region. (string, optional)
- **endpoint**: The AWS Service endpoint. (string, optional)

## Configuration for `ecs_task.register>` operator

- **ecs_task.register>**: The configuration is the same as the snake-cased [RegisterTaskDefinition API](https://docs.aws.amazon.com/AmazonECS/latest/APIReference/API_RegisterTaskDefinition.html) (map, required)

## Configuration for `ecs_task.run>` operator

The configuration is the same as the snake-cased [RunTask API](https://docs.aws.amazon.com/AmazonECS/latest/APIReference/API_RunTask.html).

In addition, the below configurations exist.

- **def**: The definition for the task. The configuration is the same as `ecs_task.register>`'s one. (map, optional)
  - **NOTE**: **task_definition** is required on the [RunTask API Doc](https://docs.aws.amazon.com/AmazonECS/latest/APIReference/API_RunTask.html), but it is not required if the **def** is defined.
- **result_s3_uri**: The S3 uri for the task result. (string, optional)
  - **NOTE**: This configuration is used by `ecs_task.result>` operator, so the result content must follow the rule.
- **timeout**: Timeout duration for the task. (`DurationParam`, default: `15m`)

## Configuration for `ecs_task.wait>` operator

- **cluster**: The short name or full ARN of the cluster that hosts the tasks. (string, required)
- **tasks**: A list of up to 100 task IDs or full ARN entries. (array of string, required)
- **timeout**: Timeout duration for the tasks. (`DurationParam`, default: `15m`)
- **condition**: The condition of tasks to wait. Available values are `"all"` or `"any"`. (string, default: `"all"`)
- **status**: The status of tasks to wait. Available values are `"PENDING"`, `"RUNNING"`, or `"STOPPED"` (string, default: `"STOPPED"`)
- **ignore_failure**: Ignore even if any tasks exit with the code except 0. (boolean, default: `false`) 

## Configuration for `ecs_task.result>` operator

- **ecs_task.result>**: S3 URI that the result is stored. (string, required)
  - **NOTE**: The result content must follow the below rule.
    - the format is json.
    - the keys are `"subtask_config"`, `"export_params"`, `"store_params"`.
    - the values are string to object map.
      - the usage follows [Digdag Python API](https://docs.digdag.io/python_api.html), [Digdag Ruby API](https://docs.digdag.io/ruby_api.html). 

# (Experimental) Scripting Operators

[digdag-operator-ecs_task](https://github.com/civitaspo/digdag-operator-ecs_task) supports some [scripting operators](https://docs.digdag.io/operators/scripting.html) such as `ecs_task.py`, `ecs_task.rb`. Originally I wanted to provide `ecs_task` as one of the implementations of `CommandExecutor` provided by digdag, but users cannot extend the current CommandExecutor as written in this issue: [\[feature-request\] Use Custom CommandExecutors](https://github.com/treasure-data/digdag/issues/901). Therefore, this plugin implements Scripting Operator on its own. Of course, the usage is the same as the Scripting Operator provided by digdag. When the issue is resolved, I will reimplement it using the `CommandExecutor` of digdag.

## Scripting Operators Common Configurations

- **sidecars**: A list of container definitions except the container for scripting operator. (array of map, optional)
    - The configuration map is the same as the snake-cased [API_ContainerDefinition](https://docs.aws.amazon.com/AmazonECS/latest/APIReference/API_ContainerDefinition.html)
- **cpu**: The number of CPU units used by the task. It can be expressed as an integer using CPU units, for example `1024`, or as a string using vCPUs, for example `1 vCPU` or `1 vcpu`, in a task definition. String values are converted to an integer indicating the CPU units when the task definition is registered. (string, optional)
    - See the docs for more info: [ECS-RegisterTaskDefinition-request-cpu](https://docs.aws.amazon.com/AmazonECS/latest/APIReference/API_RegisterTaskDefinition.html#ECS-RegisterTaskDefinition-request-cpu)
- **execution_role_arn**: The Amazon Resource Name (ARN) of the task execution role that the Amazon ECS container agent and the Docker daemon can assume. (string, optional)
- **family_prefix**: The family name prefix for a task definition. This is used if **family** is not defined. (string, default: `""`)
- **family_infix**: The family name infix for a task definition. This is used if **family** is not defined. (string, default: `"${task_name}"`)
    - The default value is replaced as below: 
        - `+` -> `_`
        - `^` -> `_`
        - `=` -> `_`
- **family_suffix**: The family name sufix for a task definition. This is used if **family** is not defined. (string, default: `""`)
- **family**: You must specify a `family` for a task definition, which allows you to track multiple versions of the same task definition. The `family` is used as a name for your task definition. Up to 255 letters (uppercase and lowercase), numbers, hyphens, and underscores are allowed. (string, default: `"${family_prefix}${family_infix}${family_suffix}"`)
- **memory**: The amount of memory (in MiB) used by the task. It can be expressed as an integer using MiB, for example `1024`, or as a string using GB, for example `1GB` or `1 GB`, in a task definition. String values are converted to an integer indicating the MiB when the task definition is registered. (string, optional)
    - See the docs for more info: [ECS-RegisterTaskDefinition-request-memory](https://docs.aws.amazon.com/AmazonECS/latest/APIReference/API_RegisterTaskDefinition.html#ECS-RegisterTaskDefinition-request-memory)
- **network_mode**: The Docker networking mode to use for the containers in the task. The valid values are `none`, `bridge`, `awsvpc`, and `host`. The default Docker network mode is `bridge`. If using the Fargate launch type, the `awsvpc` network mode is required. If using the EC2 launch type, any network mode can be used. If the network mode is set to `none`, you can't specify port mappings in your container definitions, and the task's containers do not have external connectivity. The `host` and `awsvpc` network modes offer the highest networking performance for containers because they use the EC2 network stack instead of the virtualized network stack provided by the `bridge` mode. With the `host` and `awsvpc` network modes, exposed container ports are mapped directly to the corresponding host port (for the `host` network mode) or the attached elastic network interface port (for the `awsvpc` network mode), so you cannot take advantage of dynamic host port mappings. If the network mode is `awsvpc`, the task is allocated an Elastic Network Interface, and you must specify the **network_configuration** option when you create a service or run a task with the task definition. For more information, see [Task Networking](http://docs.aws.amazon.com/AmazonECS/latest/developerguide/task-networking.html) in the Amazon Elastic Container Service Developer Guide. If the network mode is `host`, you can't run multiple instantiations of the same task on a single container instance when port mappings are used. Docker for Windows uses different network modes than Docker for Linux. When you register a task definition with Windows containers, you must not specify a network mode. (string, optional)
- **requires_compatibilities**: The launch type required by the task. If no value is specified, it defaults to `EC2`. (string, optional)
- **task_role_arn**: The short name or full Amazon Resource Name (ARN) of the IAM role that containers in this task can assume. All containers in this task are granted the permissions that are specified in this role. (string, optional)
- **volumes**: A list of volume definitions. (array of map, optional)
    - The configuration map is the same as the snake-cased [API_Volume](https://docs.aws.amazon.com/AmazonECS/latest/APIReference/API_Volume.html).
- **disable_networking**: When this parameter is `true`, networking is disabled within the container. (boolean, optional)
- **dns_search_domains**: A list of DNS search domains that are presented to the container. (array of string, optional)
- **dns_servers**: A list of DNS servers that are presented to the container. (array of string, optional)
- **docker_labels**: A key/value map of labels to add to the container. (string to string map, optional)
- **docker_security_options**: A list of strings to provide custom labels for SELinux and AppArmor multi-level security systems. This field is not valid for containers in tasks using the `Fargate` launch type. For more information, see [ECS-Type-ContainerDefinition-dockerSecurityOptions](https://docs.aws.amazon.com/AmazonECS/latest/APIReference/API_ContainerDefinition.html#ECS-Type-ContainerDefinition-dockerSecurityOptions). (array of string, optional)
- **entry_point**: The entry point that is passed to the container. (array of string, optional)
- **environments**: The environment variables to pass to a container. (string to string map, optional)
- **extra_hosts**: A list of hostnames and IP address mappings to append to the `/etc/hosts` file on the container. This parameter is not supported for Windows containers or tasks that use the `awsvpc` network mode. (string to string map, optional)
- **health_check**: The health check command and associated configuration parameters for the container. The configuration map is the same as the snake-cased [APIReference/API_HealthCheck](https://docs.aws.amazon.com/AmazonECS/latest/APIReference/API_HealthCheck.html). (map, optional)
- **hostname**: The hostname to use for your container. (string, optional)
- **image**: The image used to start a container. This string is passed directly to the Docker daemon. Images in the Docker Hub registry are available by default. Other repositories are specified with either `repository-url/image:tag` or `repository-url/image@digest`. Up to 255 letters (uppercase and lowercase), numbers, hyphens, underscores, colons, periods, forward slashes, and number signs are allowed. For more information, see [ECS-Type-ContainerDefinition-image](https://docs.aws.amazon.com/AmazonECS/latest/APIReference/API_ContainerDefinition.html#ECS-Type-ContainerDefinition-image). (string, optional)
- **interactive**: When this parameter is true, this allows you to deploy containerized applications that require `stdin` or a `tty` to be allocated. (boolean, optional)
- **links**: The `link` parameter allows containers to communicate with each other without the need for port mappings. Only supported if the network mode of a task definition is set to `bridge`. The `name:internalName` construct is analogous to `name:alias` in Docker links. Up to 255 letters (uppercase and lowercase), numbers, hyphens, and underscores are allowed. (array of string, optional)
- **linux_parameters**: Linux-specific modifications that are applied to the container, such as Linux [KernelCapabilities](https://docs.aws.amazon.com/AmazonECS/latest/APIReference/API_KernelCapabilities.html). The configuration map is the same as the snake-cased [API_LinuxParameters](https://docs.aws.amazon.com/AmazonECS/latest/APIReference/API_LinuxParameters.html). (map, optional)
- **log_configuration**: The log configuration specification for the container. For more information, see [ECS-Type-ContainerDefinition-logConfiguration](https://docs.aws.amazon.com/AmazonECS/latest/APIReference/API_ContainerDefinition.html#ECS-Type-ContainerDefinition-logConfiguration). The configuration map is the same as the snake-cased [API_LogConfiguration](https://docs.aws.amazon.com/AmazonECS/latest/APIReference/API_LogConfiguration.html). (map, optional)
- **mount_points**: The mount points for data volumes in your container. (array of map, optional)
    - The configuration map is the same as the snake-cased [API_MountPoint](https://docs.aws.amazon.com/AmazonECS/latest/APIReference/API_MountPoint.html).
- **container_name**: The name of a container. (string, default: the same as **family**)
- **port_mappings**: The list of port mappings for the container. Port mappings allow containers to access ports on the host container instance to send or receive traffic. For more informaiton, see [ECS-Type-ContainerDefinition-portMappings](https://docs.aws.amazon.com/AmazonECS/latest/APIReference/API_ContainerDefinition.html#ECS-Type-ContainerDefinition-portMappings). (array of map, optional)
    - The configuration map is the same as the snake-cased [API_PortMapping](https://docs.aws.amazon.com/AmazonECS/latest/APIReference/API_PortMapping.html).
- **privileged**: When this parameter is `true`, the container is given elevated privileges on the host container instance (similar to the `root` user). (boolean, optional)
- **pseudo_terminal**: When this parameter is `true`, a TTY is allocated. (boolean, optional)
- **readonly_root_filesystem**: When this parameter is `true`, the container is given read-only access to its root file system. (boolean, optional)
- **repository_credentials**: The private repository authentication credentials to use. The configuration map is the same as the snake-cased [API_RepositoryCredentials](https://docs.aws.amazon.com/AmazonECS/latest/APIReference/API_RepositoryCredentials.html). (map, optional)
- **system_controls**: A list of namespaced kernel parameters to set in the container. For more information, see [ECS-Type-ContainerDefinition-systemControls](https://docs.aws.amazon.com/AmazonECS/latest/APIReference/API_ContainerDefinition.html#ECS-Type-ContainerDefinition-systemControls). (array of map, optional)
    - The configuration map is the same as the snake-cased [API_SystemControl](https://docs.aws.amazon.com/AmazonECS/latest/APIReference/API_SystemControl.html).
- **ulimits**: A list of ulimits to set in the container. (array of map, optional)
    - The configuration map is the same as the snake-cased [API_Ulimit](https://docs.aws.amazon.com/AmazonECS/latest/APIReference/API_Ulimit.html).
- **user**: The user name to use inside the container. (string, optional)
- **volumes_from**: Data volumes to mount from another container. (array of map, optional)
    - The configuration map is the same as the snake-cased [API_VolumeFrom](https://docs.aws.amazon.com/AmazonECS/latest/APIReference/API_VolumeFrom.html)
- **working_directory**: The working directory in which to run commands inside the container. (string, optional)
- **cluster**: The short name or full Amazon Resource Name (ARN) of the cluster on which to run your task. (string, required)
- **count**: The number of instantiations of the specified task to place on your cluster. You can specify up to 10 tasks per call. (integer, optional)
- **group**: The name of the task group to associate with the task. The default value is the family name of the task definition (for example, family:my-family-name). (string, optional)
- **launch_type**: The launch type on which to run your task. Valid values are `EC2`, `FARGATE`. (string, optional)
- **network_configuration**: The network configuration for the task. This parameter is required for task definitions that use the `awsvpc` network mode to receive their own Elastic Network Interface, and it is not supported for other network modes. For more information, see [Task Networking](http://docs.aws.amazon.com/AmazonECS/latest/developerguide/task-networking.html) in the Amazon Elastic Container Service Developer Guide. The configuration map is the same as the snake-cased [API_NetworkConfiguration](https://docs.aws.amazon.com/AmazonECS/latest/APIReference/API_NetworkConfiguration.html). (map, optional)
- **overrides**: A list of container overrides that specify the name of a container in the specified task definition and the overrides it should receive. The configuration map is the same as the snake-cased [API_TaskOverride](https://docs.aws.amazon.com/AmazonECS/latest/APIReference/API_TaskOverride.html). (map, optional)
- **placement_constraints**: An array of placement constraint objects to use for the task. You can specify up to 10 constraints per task. (array of map, optional)
    - The configuration map is the same as the snake-cased [API_PlacementConstraint](https://docs.aws.amazon.com/AmazonECS/latest/APIReference/API_PlacementConstraint.html).
- **placement_strategy**: The placement strategy objects to use for the task. You can specify a maximum of five strategy rules per task. (array of map, optional)
    - The configuration map is the same as the snake-cased [API_PlacementStrategy](https://docs.aws.amazon.com/AmazonECS/latest/APIReference/API_PlacementStrategy.html).
- **platform_version**: The platform version on which to run your task. If one is not specified, the latest version is used by default. (string, optional)
- **started_by**: An optional tag specified when a task is started. (string, optional)

## Configuration for `ecs_task.py>` operator

- **ecs_task.py>**: Name of a method to run. The format is `[PACKAGE.CLASS.]METHOD`. (string, required)
- **workspace_s3_uri_prefix**: S3 uri prefix for using as workspace. (string, required)
- **pip_install**: packages to install before task running. (array of string, optional)

# Development

## Run an Example

### 1) build

```sh
./gradlew publish
```

Artifacts are build on local repos: `./build/repo`.

### 2) get your aws profile

```sh
aws configure
```

### 3) run an example

```sh
./example/run.sh
```

## (TODO) Run Tests

```sh
./gradlew test
```

# ChangeLog

[CHANGELOG.md](./CHANGELOG.md)

# License

[Apache License 2.0](./LICENSE.txt)

# Author

@civitaspo

