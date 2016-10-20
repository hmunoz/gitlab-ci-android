# Gitlab CI Android build runner

This build runner can build android projects and execute both unit and instrumentation tests

SDK and NDK images are included in the `sdk` directory. Dockerfile that pulls in emulators in `emulator`
directory.

With inspiration taken from:
 - https://github.com/reddit/docker-android-build/blob/master/Dockerfile
 - https://github.com/jangrewe/gitlab-ci-android
 - https://hub.docker.com/r/neroinc/fedora-android/

## Included Tools
| Tool  | Description  |
|-------|--------------|
| `android-wait-for-emulator` | Travis-ci tool available from [Github](https://github.com/travis-ci/travis-cookbooks/blob/precise-stable/ci_environment/android-sdk/files/default/android-wait-for-emulator) |
| `android-update-sdk <packages>`        | Installs the specified SDK tool packages, automatically accepting the licenses |
| `android-kill-emulators`    | Kills all running emulators |
| `android-start-emulator <image> `  | Starts the emulator that matches the name specified on the command line |
| `android-run-emulator <image> <command>` | Starts the specified emulator, runs the command (usually `./gradlew connectedAndroidTest`) and exits the emulator when done |

## .gitlab-ci.yml

The following build script will build the project, run both instrumentation and unit tests, then
publish it.

```yml
image: 51systems/gitlab-ci-android

before_script:
  - export GRADLE_USER_HOME=/cache/.gradle

stages:
  - build
  - test
  - publish

build:debug:
  stage: build
  script:
    - bash ./gradlew assembleDebug
  artifacts:
    expire_in: 1 days
    paths:
    - "**/build/outputs/**/*.apk"
    - "**/build/outputs/**/*.aar"

build:release:
  stage: build
  script: 
    - bash ./gradlew assemble
  artifacts:
    expire_in: 1 days
    paths:
    - "**/build/outputs/**/*.apk"
    - "**/build/outputs/**/*.aar"

test:unit:
  stage: test
  script:
    - bash ./gradlew test
  artifacts:
    name: "Tests-${CI_BUILD_NAME}_${CI_BUILD_REF_NAME}_${CI_BUILD_REF}"
    when: always
    expire_in: 1 weeks
    paths:
      - "**/build/reports/tests"

test:instrumentation:21:
  stage: test
  image: 51systems/gitlab-ci-android-emulator-21
  script:
    - android-run-emulator nexus5_21 bash ./gradlew connectedAndroidTest
  artifacts:
    name: "Tests-Instrumentation-21-${CI_BUILD_NAME}_${CI_BUILD_REF_NAME}_${CI_BUILD_REF}"
    when: always
    expire_in: 1 weeks
    paths:
      - "**/build/reports/androidTests"

publish:release:
  stage: publish
  script:
    - bash ./gradlew publish

```