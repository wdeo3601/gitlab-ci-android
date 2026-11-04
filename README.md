# gitlab-ci-android(fork)

Docker Hub:https://hub.docker.com/r/wdeo3601/gitlab-ci-android

上游:`https://github.com/inovex/gitlab-ci-android`(Docker Hub:`https://hub.docker.com/r/inovex/gitlab-ci-android`)

基于上游升级了基座与 JDK、固化了项目所需的 SDK/NDK 组件的自建 CI 镜像。

## 镜像内容

| 项 | 内容 |
|---|---|
| 基础镜像 | ubuntu:26.04 LTS(上游为 20.04,已 EOL) |
| JDK | 17 / 21 / 25 / 26(Ubuntu 官方源;上游的 8/11 已移除) |
| 默认 JVM | `JAVA_HOME=/usr/lib/jvm/java-25-openjdk-amd64`(配合 Gradle 9.5 的 daemon JVM criteria) |
| build-tools | 34.0.0 / 35.0.0 / 36.0.0 |
| platforms | android-34 / 35 / 36 |
| NDK | 28.2.13676358(装在 `/sdk/ndk/`,即 CI 探测路径;上游钉在 `/ndk` 的 r25c 已移除) |

## JAVA_HOME 用法

```
before_script:
  - export JAVA_HOME=/usr/lib/jvm/java-25-openjdk-amd64   # Gradle 9.1+
  - export JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64
  - export JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64
  - export JAVA_HOME=/usr/lib/jvm/java-26-openjdk-amd64   # Gradle 尚未支持,预留
```

## 在 CI 中使用

```
image: wdeo3601/gitlab-ci-android:26.04-jdk25-v1
```

`JAVA_HOME` 按目标工程的 Gradle 版本在 `before_script` 中自行 export(见上方用法)。

## 相对上游的改动清单

- 基础镜像 ubuntu:20.04 → 26.04 LTS
- JDK:17/21/25/26 官方源(移除 8/11)
- NDK:改由 `pkg.txt` 固化 `ndk;28.2.13676358` 到 `/sdk/ndk`(CI 探测路径),移除上游 `/ndk` 的 r25c 下载块
- `pkg.txt` 增补 `platforms;android-36`、`build-tools;36.0.0`(compileSdk 36 所需)
- 移除 Google repo 工具安装(本项目不使用,且其钉死的 SHA256 随上游脚本更新会失效)
- 26.04 适配:`localedef` 去掉 `-A` 参数(glibc 不再分发 locale.alias)

## 修改与发布流程

1. 修改 `Dockerfile` 或 `pkg.txt`;
2. **tag 尾部版本号 +1**(如 `26.04-jdk25-v1` → `26.04-jdk25-v2`),旧 tag 永不覆盖,保证 CI 引用的镜像内容可追溯;
3. 构建并推送(先 `docker login`):

```
docker build --platform linux/amd64 -t <DockerHub用户名>/gitlab-ci-android:<新tag> .
docker push <DockerHub用户名>/gitlab-ci-android:<新tag>
```

4. 在 CI 仓库把 `image:` 升到新 tag——镜像升级是显式、可 review、可回滚的动作;
5. Runner 出网拉取 Docker Hub 失败时的应急通道:`docker save | gzip` → 上传服务器 → `docker load`(runner 拉取策略为 `if-not-present`,本机存在即直接使用)。

## 上游示例(依赖缓存的 CI 配置,镜像已替换为本 fork)

```
image: wdeo3601/gitlab-ci-android:26.04-jdk25-v1

stages:
- release

variables:
  GRADLE_OPTS: "-Dorg.gradle.daemon=false"

before_script:
- export GRADLE_USER_HOME=$(pwd)/.gradle
- chmod +x ./gradlew

cache:
  key: ${CI_PROJECT_ID}
  paths:
  - .gradle/

build:
    stage: release
    script:
        - ./gradlew clean assembleRelease
    artifacts:
        expire_in: 2 weeks
        paths:
            - app/build/outputs/apk/*.apk
    only:
        - develop
```

## License

```
Gitlab CI Android
Copyright (c) 2024 inovex GmbH (https://www.inovex.de)

Modifications Copyright (c) 2026 wdeo3601

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
```
