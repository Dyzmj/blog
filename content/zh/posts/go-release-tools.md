---
title : 'GoReleaser：自动化你的软件发布'
description:
ShowRssButtonInSectionTermList: true
cover.image:
date : 2023-09-16T16:07:39+08:00
draft : false
showtoc: true
tocopen: true
author: ["熊鑫伟", "Me"]
keywords: []
tags:
  - blog
categories:
  - Development
  - Blog
---

GoReleaser 的目标是自动化发布软件时的大部分繁琐工作，通过使用合理的默认值并使最常见的用例变得简单。

## 准备工作：

- `.goreleaser.yaml` 文件：包含所有配置信息。（有关更多信息，请参阅 [自定义](https://goreleaser.com/customization/)）
- 干净的工作树：确保代码是最新的，并且已经提交了所有改动。
- 符合 SemVer 的版本号（例如 `10.21.34-prerelease+buildmeta`）

## GoReleaser 的运行步骤：

GoReleaser 的运行主要分为以下四个步骤：

1. **defaulting**：为每个步骤配置合理的默认值
2. **building**：构建二进制文件、档案、包、Docker 镜像等
3. **releasing**：将版本发布到配置的 SCM、Docker 注册表、blob 存储等
4. **announcing**：向配置的频道宣布您的发布

使用 `-like` 标志可能会跳过某些步骤，如 `--skip-foo`

## 快速开始

首先，运行 [init](https://goreleaser.com/cmd/goreleaser_init/) 命令来创建示例的 `.goreleaser.yaml` 文件：

```
goreleaser init
```

然后，我们运行一个“仅限本地”版本，看看它是否可以使用 [release](https://goreleaser.com/cmd/goreleaser_release/) 命令运行：

```
goreleaser release --snapshot --rm-dist
```

此时，您可以 [自定义](https://goreleaser.com/customization/) 生成的 `.goreleaser.yaml` 文件，或保持原样，这取决于您。最佳做法是将 `.goreleaser.yaml` 文件放入版本控制系统中。

您还可以使用 GoReleaser 为给定的 GOOS/GOARCH [构建二进制文件](https://goreleaser.com/cmd/goreleaser_build/)，这对于本地开发非常有用：

```
goreleaser build --single-target
```

准备 GitHub 的 Token：

```
export GITHUB_TOKEN="YOUR_GH_TOKEN"
```

GoReleaser 将使用您存储库的最新 [Git 标签](https://git-scm.com/book/en/v2/Git-Basics-Tagging)。

创建一个标签并将其推送到 GitHub：

```
git push origin v0.1.0
```

现在，您可以在项目的根目录运行 GoReleaser：

```
goreleaser release
```

## 在 GitHub Actions 中使用 GoReleaser

GoReleaser 还可以通过 [GitHub Actions](https://github.com/features/actions) 在我们的官方 [GoReleaser Action](https://github.com/goreleaser/goreleaser-action) 中使用。

您可以通过将 YAML 配置放入 `.github/workflows/release.yml` 文件中。

```bash
bashcodename: goreleaser

on:
  push:
    # 只对标签进行运行
    tags:
      - '*'

permissions:
  contents: write
  # packages: write
  # issues: write

jobs:
  goreleaser:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
        with:
          fetch-depth: 0
      - run: git fetch --force --tags
      - uses: actions/setup-go@v4
        with:
          go-version: stable
      # 更多设置可能需要，如 Docker 登录、GPG 等。这些都取决于您的需求。
      - uses: goreleaser/goreleaser-action@v4
        with:
          # 可以选择 'goreleaser'（默认）或 'goreleaser-pro'
          distribution: goreleaser
          version: latest
          args: release --rm-dist
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          # 如果你正在使用 'goreleaser-pro' 发行版，你需要 GoReleaser Pro 的密钥：
          # GORELEASER_KEY: ${{ secrets.GORELEASER_KEY }}
```

GoReleaser 需要以下 [权限](https://docs.github.com/en/actions/reference/authentication-in-a-workflow#permissions-for-the-github_token)：

- ```
  contents: write
  ```

  ，如果你打算：

  - [将档案上传为 GitHub Releases](https://goreleaser.com/customization/release/)，或
  - 发布到 [Homebrew](https://goreleaser.com/customization/homebrew/) 或 [Scoop](https://goreleaser.com/customization/scoop/)（假设它是同一存储库的一部分）

- `packages: write`，如果你打算 [将 Docker 镜像推送到](https://goreleaser.com/customization/docker/) GitHub

- `issues: write`，如果你使用了 [里程碑关闭功能](https://goreleaser.com/customization/milestone/)

`GITHUB_TOKEN` 权限 [只限于](https://help.github.com/en/actions/configuring-and-managing-workflows/authenticating-with-the-github_token#about-the-github_token-secret) 包含你的工作流的存储库。

如果你需要将 Homebrew Tap 推送到另一个存储库，那么你必须创建一个有权访问的个人访问令牌，并将其添加为存储库的秘密。如果你创建了一个名为 `GH_PAT` 的秘密，那么步骤将如下：

```
yaml
```

```yaml
      - name: Run GoReleaser
        uses: goreleaser/goreleaser-action@v4
        with:
          version: latest
          args: release --rm-dist
        env:
          GITHUB_TOKEN: ${{ secrets.GH_PAT }}
```



## 定制化需求

GoReleaser 可以通过调整`.goreleaser.yaml`文件来定制。

[goreleaser init](https://goreleaser.com/cmd/goreleaser_init/)您可以通过运行或从头开始生成示例配置。

您还可以通过运行来检查您的配置是否有效[goreleaser check](https://goreleaser.com/cmd/goreleaser_check/)，这会告诉您是否使用了已弃用或无效的选项。

### 名称模板

| 钥匙                   | 描述                                                         |
| ---------------------- | ------------------------------------------------------------ |
| `.ProjectName`         | 项目名称                                                     |
| `.Version`             | 正在发布的版本 ([详情](https://goreleaser.com/customization/templates/#fn:version-prefix)) |
| `.Branch`              | 当前的 git 分支                                              |
| `.PrefixedTag`         | 以 monorepo 配置标签前缀为前缀的当前 git 标签（如果有）      |
| `.Tag`                 | 当前的 git 标签                                              |
| `.PrefixedPreviousTag` | 之前的 git 标签带有 monorepo 配置标签前缀（如果有）          |
| `.PreviousTag`         | 之前的 git 标签，如果没有之前的标签则为空                    |
| `.ShortCommit`         | git 提交短哈希                                               |
| `.FullCommit`          | git 提交完整哈希值                                           |
| `.Commit`              | git 提交哈希（已弃用）                                       |
| `.CommitDate`          | RFC 3339 格式的 UTC 提交日期                                 |
| `.CommitTimestamp`     | Unix 格式的 UTC 提交日期                                     |
| `.GitURL`              | git 远程 URL                                                 |
| `.IsGitDirty`          | 当前 git 状态是否脏。自 v1.19 起。                           |
| `.Major`               | 版本 ([详情](https://goreleaser.com/customization/templates/#fn:tag-is-semver)) |
| `.Minor`               | 版本 ([详情](https://goreleaser.com/customization/templates/#fn:tag-is-semver)) |
| `.Patch`               | 补丁部分 ([详情](https://goreleaser.com/customization/templates/#fn:tag-is-semver)) |
| `.Prerelease`          | 版本的预发行部分，例如beta ([详情](https://goreleaser.com/customization/templates/#fn:tag-is-semver)) |
| `.RawVersion`          | 组成 `{Major}.{Minor}.{Patch}` ([详情](https://goreleaser.com/customization/templates/#fn:tag-is-semver)) |
| `.ReleaseNotes`        | 生成的发行说明，在执行变更日志步骤后可用                     |
| `.IsDraft`             | true if `release.draft` 在配置中设置，false否则。自 v1.17 起。 |
| `.IsSnapshot`          | true 如果 `--snapshot` 已设置，false否则                     |
| `.IsNightly`           | true 如果 `--nightly` 已设置，false否则                      |
| `.Env`                 | 包含系统环境变量的映射                                       |
| `.Date`                | RFC 3339 格式的当前 UTC 日期                                 |
| `.Now`                 | 当前 UTC 日期作为 `time.Time` 结构，允许所有 `time.Time` 功能（例如 `{{ .Now.Format "2006" }}` ）。自 v1.17 起。 |
| `.Timestamp`           | Unix 格式的当前 UTC 时间                                     |
| `.ModulePath`          | go 模块路径，如报告所示 `go list -m`                         |
| `incpatch "v1.2.4"`    | 补丁 ([详情](https://goreleaser.com/customization/templates/#fn:panic-if-not-semver)) |
| `incminor "v1.2.4"`    | 次要版本 ([详情](https://goreleaser.com/customization/templates/#fn:panic-if-not-semver)) |
| `incmajor "v1.2.4"`    | 增加给定版本 ([详情](https://goreleaser.com/customization/templates/#fn:panic-if-not-semver)) |
| `.ReleaseURL`          | 当前版本下载地址 ([详情](https://goreleaser.com/customization/templates/#fn:scm-release-url)) |
| `.Summary`             | git 摘要，例如 `v1.0.0-10-g34f56g3` ([详情](https://goreleaser.com/customization/templates/#fn:git-summary)) |
| `.PrefixedSummary`     | 以 monorepo 配置标签前缀为前缀的 git 摘要（如果有）          |
| `.TagSubject`          | 带注释的标签消息主题，或者它指出的提交的消息主题 ([详情](https://goreleaser.com/customization/templates/#fn:git-tag-subject))。从 v1.2 开始。 |
| `.TagContents`         | 带注释的标签消息，或者它指出的提交消息 ([详情](https://goreleaser.com/customization/templates/#fn:git-tag-body)) . 从 v1.2 开始。 |
| `.TagBody`             | 带注释的标签消息正文，或其指出的提交的消息正文 ([详情](https://goreleaser.com/customization/templates/#fn:git-tag-body))。从 v1.2 开始。 |
| `.Runtime.Goos`        | 相当于 `runtime.GOOS`. 从 v1.5 开始。                        |
| `.Runtime.Goarch`      | 相当于 `runtime.GOARCH`. 从 v1.5 开始。                      |
| `.Artifacts`           | 当前工件列表。字段见下表。自 v1.16-pro 起。                  |



### 配置选项

```yaml
# Default: './dist'
dist: _output/dist

git:
  # What should be used to sort tags when gathering the current and previous
  # tags if there are more than one tag in the same commit.
  #
  # Default: '-version:refname'
  tag_sort: -version:creatordate

  # What should be used to specify prerelease suffix while sorting tags when gathering
  # the current and previous tags if there are more than one tag in the same commit.
  #
  # Since: v1.17
  prerelease_suffix: "-"
```



### 构建选项

可以通过多种方式定制构建。您可以指定构建哪些`GOOS`,`GOARCH`和`GOARM`二进制文件（GoReleaser 将生成所有组合的矩阵），并且您可以更改二进制文件的名称、标志、环境变量、挂钩等。

**builds 是配置文件中最重要的一个选项：**

```yaml
# .goreleaser.yaml
builds:
  # You can have multiple builds defined as a yaml list
  - #
    # ID of the build.
    #
    # Default: Binary name
    id: "my-build"

    # Path to main.go file or main package.
    # Notice: when used with `gomod.proxy`, this must be a package.
    #
    # Default is `.`.
    main: ./cmd/my-app

    # Binary name.
    # Can be a path (e.g. `bin/app`) to wrap the binary in a directory.
    #
    # Default: Project directory name
    binary: program

    # Custom flags.
    #
    # Templates: allowed
    flags:
      - -tags=dev
      - -v

    # Custom asmflags.
    #
    # Templates: allowed
    asmflags:
      - -D mysymbol
      - all=-trimpath={{.Env.GOPATH}}

    # Custom gcflags.
    #
    # Templates: allowed
    gcflags:
      - all=-trimpath={{.Env.GOPATH}}
      - ./dontoptimizeme=-N

    # Custom ldflags.
    #
    # Default: '-s -w -X main.version={{.Version}} -X main.commit={{.Commit}} -X main.date={{.Date}} -X main.builtBy=goreleaser'
    # Templates: allowed
    ldflags:
      - -s -w -X main.build={{.Version}}
      - ./usemsan=-msan

    # Custom Go build mode.
    #
    # Valid options:
    # - `c-shared`
    # - `c-archive`
    #
    # Since: v1.13
    buildmode: c-shared

    # Custom build tags templates.
    tags:
      - osusergo
      - netgo
      - static_build
      - feature

    # Custom environment variables to be set during the builds.
    # Invalid environment variables will be ignored.
    #
    # Default: os.Environ() ++ env config section
    # Templates: allowed (since v1.14)
    env:
      - CGO_ENABLED=0
      # complex, templated envs (v1.14+):
      - >-
        {{- if eq .Os "darwin" }}
          {{- if eq .Arch "amd64"}}CC=o64-clang{{- end }}
          {{- if eq .Arch "arm64"}}CC=aarch64-apple-darwin20.2-clang{{- end }}
        {{- end }}
        {{- if eq .Os "windows" }}
          {{- if eq .Arch "amd64" }}CC=x86_64-w64-mingw32-gcc{{- end }}
        {{- end }}

    # GOOS list to build for.
    # For more info refer to: https://golang.org/doc/install/source#environment
    #
    # Default: [ 'darwin', 'linux', 'windows' ]
    goos:
      - freebsd
      - windows

    # GOARCH to build for.
    # For more info refer to: https://golang.org/doc/install/source#environment
    #
    # Default: [ '386', 'amd64', 'arm64' ]
    goarch:
      - amd64
      - arm
      - arm64

    # GOARM to build for when GOARCH is arm.
    # For more info refer to: https://golang.org/doc/install/source#environment
    #
    # Default: [ 6 ]
    goarm:
      - 6
      - 7

    # GOAMD64 to build when GOARCH is amd64.
    # For more info refer to: https://golang.org/doc/install/source#environment
    #
    # Default: [ 'v1' ]
    goamd64:
      - v2
      - v3

    # GOMIPS and GOMIPS64 to build when GOARCH is mips, mips64, mipsle or mips64le.
    # For more info refer to: https://golang.org/doc/install/source#environment
    #
    # Default: [ 'hardfloat' ]
    gomips:
      - hardfloat
      - softfloat

    # List of combinations of GOOS + GOARCH + GOARM to ignore.
    ignore:
      - goos: darwin
        goarch: 386
      - goos: linux
        goarch: arm
        goarm: 7
      - goarm: mips64
      - gomips: hardfloat
      - goamd64: v4

    # Optionally override the matrix generation and specify only the final list
    # of targets.
    #
    # Format is `{goos}_{goarch}` with their respective suffixes when
    # applicable: `_{goarm}`, `_{goamd64}`, `_{gomips}`.
    #
    # Special values:
    # - go_118_first_class: evaluates to the first-class ports of go1.18.
    # - go_first_class: evaluates to latest stable go first-class ports,
    #   currently same as 1.18.
    #
    # This overrides `goos`, `goarch`, `goarm`, `gomips`, `goamd64` and
    # `ignores`.
    targets:
      # Since: v1.9
      - go_first_class
      # Since: v1.9
      - go_118_first_class
      - linux_amd64_v1
      - darwin_arm64
      - linux_arm_6

    # Set a specific go binary to use when building.
    # It is safe to ignore this option in most cases.
    #
    # Default is "go"
    gobinary: "go1.13.4"

    # Sets the command to run to build.
    # Can be useful if you want to build tests, for example,
    # in which case you can set this to "test".
    # It is safe to ignore this option in most cases.
    #
    # Default: build.
    # Since: v1.9
    command: test

    # Set the modified timestamp on the output binary, typically
    # you would do this to ensure a build was reproducible. Pass
    # empty string to skip modifying the output.
    #
    # Templates: allowed.
    mod_timestamp: "{{ .CommitTimestamp }}"

    # Hooks can be used to customize the final binary,
    # for example, to run generators.
    #
    # Templates: allowed
    hooks:
      pre: rice embed-go
      post: ./script.sh {{ .Path }}

    # If true, skip the build.
    # Useful for library projects.
    skip: false

    # By default, GoReleaser will create your binaries inside
    # `dist/${BuildID}_${BuildTarget}`, which is a unique directory per build
    # target in the matrix.
    # You can set subdirs within that folder using the `binary` property.
    #
    # However, if for some reason you don't want that unique directory to be
    # created, you can set this property.
    # If you do, you are responsible for keeping different builds from
    # overriding each other.
    no_unique_dist_dir: true

    # By default, GoReleaser will check if the main filepath has a main
    # function.
    # This can be used to skip that check, in case you're building tests, for
    # example.
    #
    # Since: v1.9
    no_main_check: true

    # Path to project's (sub)directory containing Go code.
    # This is the working directory for the Go build command(s).
    # If dir does not contain a `go.mod` file, and you are using `gomod.proxy`,
    # produced binaries will be invalid.
    # You would likely want to use `main` instead of this.
    #
    # Default: '.'
    dir: go

    # Builder allows you to use a different build implementation.
    # This is a GoReleaser Pro feature.
    # Valid options are: `go` and `prebuilt`.
    #
    # Default: 'go'
    builder: prebuilt

    # Overrides allows to override some fields for specific targets.
    # This can be specially useful when using CGO.
    # Note: it'll only match if the full target matches.
    #
    # Since: v1.5
    overrides:
      - goos: darwin
        goarch: arm64
        goamd64: v1
        goarm: ""
        gomips: ""
        ldflags:
          - foo
        tags:
          - bar
        asmflags:
          - foobar
        gcflags:
          - foobaz
        env:
          - CGO_ENABLED=1
```

**包含多个二进制文件的构建：**

```yaml
# .goreleaser.yaml
builds:
  - main: ./cmd/cli
    id: "cli"
    binary: cli
    goos:
      - linux
      - darwin
      - windows

  - main: ./cmd/worker
    id: "worker"
    binary: worker
    goos:
      - linux
      - darwin
      - windows

  - main: ./cmd/tracker
    id: "tracker"
    binary: tracker
    goos:
      - linux
      - darwin
      - windows
```

二进制名称字段支持[模板化](https://goreleaser.com/customization/templates/)。公开了以下构建详细信息：

| Key     | Description                     |
| ------- | ------------------------------- |
| .Os     | GOOS                            |
| .Arch   | GOARCH                          |
| .Arm    | GOARM                           |
| .Ext    | Extension, e.g. .exe            |
| .Target | Build target, e.g. darwin_amd64 |

您可以通过`{{ .Env.VARIABLE_NAME }}`在模板中使用来做到这一点，例如：

```yaml
# .goreleaser.yaml
builds:
  - ldflags:
   - -s -w -X "main.goversion={{.Env.GOVERSION}}"
```

然后你可以运行：

`GOVERSION=$(go version) goreleaser`

## build hooks

pre 和 post 挂钩都 **针对每个构建目标** 运行，无论这些目标是通过操作系统和架构矩阵生成还是显式定义。

除了上面所示的简单声明之外，还可以声明多个挂钩，以帮助保持不同构建环境之间配置的可重用性。

```yaml
# .goreleaser.yaml
builds:
  - id: "with-hooks"
    targets:
      - "darwin_amd64"
      - "windows_amd64"
    hooks:
      pre:
        - first-script.sh
        - second-script.sh
      post:
        - upx "{{ .Path }}"
        - codesign -project="{{ .ProjectName }}" "{{ .Path }}"
```

每个钩子还可以有自己的工作目录和环境变量：

```yaml
# .goreleaser.yaml
builds:
  - id: "with-hooks"
    targets:
      - "darwin_amd64"
      - "windows_amd64"
    hooks:
      pre:
        - cmd: first-script.sh
          dir:
            "{{ dir .Dist}}"
            # Always print command output, otherwise only visible in debug mode.
            # Since: v1.5
          output: true
          env:
            - HOOK_SPECIFIC_VAR={{ .Env.GLOBAL_VAR }}
        - second-script.sh
```

钩子的所有属性（`cmd`、`dir`和`env`）都支持使用具有可用二进制工件的钩子[进行模板化](https://goreleaser.com/customization/templates/)（因为它们在构建*后运行）。*此外，以下构建详细信息也会暴露给和钩子：`postprepost`

| Key     | Description                          |
| ------- | ------------------------------------ |
| .Name   | Filename of the binary, e.g. bin.exe |
| .Ext    | Extension, e.g. .exe                 |
| .Path   | Absolute path to the binary          |
| .Target | Build target, e.g. darwin_amd64      |

环境变量按以下顺序继承和覆盖：

- 全局 ( `env`)
- 构建 ( `builds[].env`)
- 钩子 (`builds[].hooks.pre[].env`和`builds[].hooks.post[].env`)

### 模块

如果您使用带有 go 模块或 vgo 的 Go 1.11+，当 GoReleaser 运行时，它可能会尝试下载依赖项。由于多个构建并行运行，因此很可能会失败。

您可以通过`go mod tidy`在调用之前运行`goreleaser`或添加一个在文件上执行此操作的[挂钩](https://goreleaser.com/customization/hooks)`.goreleaser.yaml`来解决此问题：

```yaml
# .goreleaser.yaml
before:
  hooks:
    - go mod tidy
# rest of the file...
```

## archives

`README`构建的二进制文件将与和文件一起归档`LICENSE`到一个`tar.gz`文件中。在此`archives`部分中，您可以自定义存档名称、其他文件和格式。

```yaml
# .goreleaser.yaml
archives:
  - #
    # ID of this archive.
    #
    # Default: 'default'
    id: my-archive

    # Builds reference which build instances should be archived in this archive.
    builds:
      - default

    # Archive format. Valid options are `tar.gz`, `tgz`, `tar.xz`, `txz`, tar`, `gz`, `zip` and `binary`.
    # If format is `binary`, no archives are created and the binaries are instead
    # uploaded directly.
    #
    # Default: 'tar.gz'
    format: zip

    # This will create an archive without any binaries, only the files are there.
    # The name template must not contain any references to `Os`, `Arch` and etc, since the archive will be meta.
    #
    # Since: v1.9
    # Templates: allowed
    meta: true

    # Archive name.
    #
    # Default:
    # - if format is `binary`:
    #   - `{{ .Binary }}_{{ .Version }}_{{ .Os }}_{{ .Arch }}{{ with .Arm }}v{{ . }}{{ end }}{{ with .Mips }}_{{ . }}{{ end }}{{ if not (eq .Amd64 "v1") }}{{ .Amd64 }}{{ end }}`
    # - if format is anything else:
    #   - `{{ .ProjectName }}_{{ .Version }}_{{ .Os }}_{{ .Arch }}{{ with .Arm }}v{{ . }}{{ end }}{{ with .Mips }}_{{ . }}{{ end }}{{ if not (eq .Amd64 "v1") }}{{ .Amd64 }}{{ end }}`
    # Templates: allowed
    name_template: "{{ .ProjectName }}_{{ .Version }}_{{ .Os }}_{{ .Arch }}"

    # Sets the given file info to all the binaries included from the `builds`.
    #
    # Default: copied from the source binary.
    # Since: v1.14
    builds_info:
      group: root
      owner: root
      mode: 0644
      # format is `time.RFC3339Nano`
      mtime: 2008-01-02T15:04:05Z

    # Set this to true if you want all files in the archive to be in a single directory.
    # If set to true and you extract the archive 'goreleaser_Linux_arm64.tar.gz',
    # you'll get a folder 'goreleaser_Linux_arm64'.
    # If set to false, all files are extracted separately.
    # You can also set it to a custom folder name (templating is supported).
    wrap_in_directory: true

    # If set to true, will strip the parent directories away from binary files.
    #
    # This might be useful if you have your binary be built with a subdir for some reason, but do no want that subdir inside the archive.
    #
    # Since: v1.11
    strip_parent_binary_folder: true

    # This will make the destination paths be relative to the longest common
    # path prefix between all the files matched and the source glob.
    # Enabling this essentially mimic the behavior of nfpm's contents section.
    # It will be the default by June 2023.
    #
    # Since: v1.14
    rlcp: true

    # Can be used to change the archive formats for specific GOOSs.
    # Most common use case is to archive as zip on Windows.
    format_overrides:
      - goos: windows
        format: zip

    # Additional files/globs you want to add to the archive.
    #
    # Default: [ 'LICENSE*', 'README*', 'CHANGELOG', 'license*', 'readme*', 'changelog']
    # Templates: allowed
    files:
      - LICENSE.txt
      - README_{{.Os}}.md
      - CHANGELOG.md
      - docs/*
      - design/*.png
      - templates/**/*
      # a more complete example, check the globbing deep dive below
      - src: "*.md"
        dst: docs

        # Strip parent folders when adding files to the archive.
        strip_parent: true

        # File info.
        # Not all fields are supported by all formats available formats.
        #
        # Default: copied from the source file
        info:
          # Templates: allowed (since v1.14)
          owner: root

          # Templates: allowed (since v1.14)
          group: root

          # Must be in time.RFC3339Nano format.
          #
          # Templates: allowed (since v1.14)
          mtime: "{{ .CommitDate }}"

          # File mode.
          mode: 0644

    # Additional templated files to add to the archive.
    # Those files will have their contents pass through the template engine,
    # and its results will be added to the archive.
    #
    # Since: v1.17 (pro)
    # This feature is only available in GoReleaser Pro.
    # Templates: allowed
    templated_files:
      # a more complete example, check the globbing deep dive below
      - src: "LICENSE.md.tpl"
        dst: LICENSE.md

        # File info.
        # Not all fields are supported by all formats available formats.
        #
        # Default: copied from the source file
        info:
          # Templateable (since v1.14.0)
          owner: root

          # Templateable (since v1.14.0)
          group: root

          # Must be in time.RFC3339Nano format.
          # Templateable (since v1.14.0)
          mtime: "{{ .CommitDate }}"

          # File mode.
          mode: 0644

    # Before and after hooks for each archive.
    # Skipped if archive format is binary.
    # This feature is only available in GoReleaser Pro.
    hooks:
      before:
        - make clean # simple string
        - cmd: go generate ./... # specify cmd
        - cmd: go mod tidy
          output: true # always prints command output
          dir: ./submodule # specify command working directory
        - cmd: touch {{ .Env.FILE_TO_TOUCH }}
          env:
            - "FILE_TO_TOUCH=something-{{ .ProjectName }}" # specify hook level environment variables

      after:
        - make clean
        - cmd: cat *.yaml
          dir: ./submodule
        - cmd: touch {{ .Env.RELEASE_DONE }}
          env:
            - "RELEASE_DONE=something-{{ .ProjectName }}" # specify hook level environment variables

    # Disables the binary count check.
    allow_different_binary_count: true
```

## linux 软件包

GoReleaser 可以连接到[nfpm](https://github.com/goreleaser/nfpm)以生成和发布 `.deb`、`.rpm`、`.apk` 和 `Archlinux` 软件包。

```bash
# .goreleaser.yaml
nfpms:
  # note that this is an array of nfpm configs
  - #
    # ID of the nfpm config, must be unique.
    #
    # Default: 'default'
    id: foo

    # Name of the package.
    # Default: ProjectName
    # Templates: allowed (since v1.18)
    package_name: foo

    # You can change the file name of the package.
    #
    # Default: '{{ .PackageName }}_{{ .Version }}_{{ .Os }}_{{ .Arch }}{{ with .Arm }}v{{ . }}{{ end }}{{ with .Mips }}_{{ . }}{{ end }}{{ if not (eq .Amd64 "v1") }}{{ .Amd64 }}{{ end }}'
    # Templates: allowed
    file_name_template: "{{ .ConventionalFileName }}"

    # Build IDs for the builds you want to create NFPM packages for.
    # Defaults empty, which means no filtering.
    builds:
      - foo
      - bar

    # Your app's vendor.
    vendor: Drum Roll Inc.

    # Your app's homepage.
    homepage: https://example.com/

    # Your app's maintainer (probably you).
    maintainer: Drummer <drum-roll@example.com>

    # Your app's description.
    description: |-
      Drum rolls installer package.
      Software to create fast and easy drum rolls.

    # Your app's license.
    license: Apache 2.0

    # Formats to be generated.
    formats:
      - apk
      - deb
      - rpm
      - termux.deb # Since: v1.11
      - archlinux # Since: v1.13

    # Umask to be used on files without explicit mode set. (overridable)
    #
    # Default: 0o002 (will remove world-writable permissions)
    # Since: v1.19
    umask: 0o002

    # Packages your package depends on. (overridable)
    dependencies:
      - git
      - zsh

    # Packages it provides. (overridable)
    #
    # Since: v1.11
    provides:
      - bar

    # Packages your package recommends installing. (overridable)
    recommends:
      - bzr
      - gtk

    # Packages your package suggests installing. (overridable)
    suggests:
      - cvs
      - ksh

    # Packages that conflict with your package. (overridable)
    conflicts:
      - svn
      - bash

    # Packages it replaces. (overridable)
    replaces:
      - fish

    # Path that the binaries should be installed.
    # Default: '/usr/bin'
    bindir: /usr/bin

    # Version Epoch.
    # Default: extracted from `version` if it is semver compatible
    epoch: 2

    # Version Prerelease.
    # Default: extracted from `version` if it is semver compatible
    prerelease: beta1

    # Version Metadata (previously deb.metadata).
    # Setting metadata might interfere with version comparisons depending on the
    # packager.
    #
    # Default: extracted from `version` if it is semver compatible
    version_metadata: git

    # Version Release.
    release: 1

    # Section.
    section: default

    # Priority.
    priority: extra

    # Makes a meta package - an empty package that contains only supporting
    # files and dependencies.
    # When set to `true`, the `builds` option is ignored.
    #
    # Default: false
    meta: true

    # Changelog YAML file, see: https://github.com/goreleaser/chglog
    #
    # You can use goreleaser/chglog to create the changelog for your project,
    # pass that changelog yaml file to GoReleaser,
    # and it should in turn setup it accordingly for the given available
    # formats (deb and rpm at the moment).
    #
    # Experimental.
    # Since: v1.11
    changelog: ./foo.yml

    # Contents to add to the package.
    # GoReleaser will automatically add the binaries.
    contents:
      # Basic file that applies to all packagers
      - src: path/to/foo
        dst: /usr/bin/foo

      # This will add all files in some/directory or in subdirectories at the
      # same level under the directory /etc. This means the tree structure in
      # some/directory will not be replicated.
      - src: some/directory/
        dst: /etc

      # This will replicate the directory structure under some/directory at
      # /etc, using the "tree" type.
      #
      # Since: v1.17
      # Templates: allowed
      - src: some/directory/
        dst: /etc
        type: tree

      # Simple config file
      - src: path/to/foo.conf
        dst: /etc/foo.conf
        type: config

      # Simple symlink.
      # Corresponds to `ln -s /sbin/foo /usr/local/bin/foo`
      - src: /sbin/foo
        dst: /usr/bin/foo
        type: "symlink"

      # Corresponds to `%config(noreplace)` if the packager is rpm, otherwise it
      # is just a config file
      - src: path/to/local/bar.conf
        dst: /etc/bar.conf
        type: "config|noreplace"

      # The src and dst attributes also supports name templates
      - src: path/{{ .Os }}-{{ .Arch }}/bar.conf
        dst: /etc/foo/bar-{{ .ProjectName }}.conf

    # Additional templated contents to add to the archive.
    # Those files will have their contents pass through the template engine,
    # and its results will be added to the package.
    #
    # Since: v1.17 (pro)
    # This feature is only available in GoReleaser Pro.
    # Templates: allowed
    templated_contents:
      # a more complete example, check the globbing deep dive below
      - src: "LICENSE.md.tpl"
        dst: LICENSE.md

      # These files are not actually present in the package, but the file names
      # are added to the package header. From the RPM directives documentation:
      #
      # "There are times when a file should be owned by the package but not
      # installed - log files and state files are good examples of cases you
      # might desire this to happen."
      #
      # "The way to achieve this, is to use the %ghost directive. By adding this
      # directive to the line containing a file, RPM will know about the ghosted
      # file, but will not add it to the package."
      #
      # For non rpm packages ghost files are ignored at this time.
      - dst: /etc/casper.conf
        type: ghost
      - dst: /var/log/boo.log
        type: ghost

      # You can use the packager field to add files that are unique to a
      # specific packager
      - src: path/to/rpm/file.conf
        dst: /etc/file.conf
        type: "config|noreplace"
        packager: rpm
      - src: path/to/deb/file.conf
        dst: /etc/file.conf
        type: "config|noreplace"
        packager: deb
      - src: path/to/apk/file.conf
        dst: /etc/file.conf
        type: "config|noreplace"
        packager: apk

      # Sometimes it is important to be able to set the mtime, mode, owner, or
      # group for a file that differs from what is on the local build system at
      # build time.
      - src: path/to/foo
        dst: /usr/local/foo
        file_info:
          mode: 0644
          mtime: 2008-01-02T15:04:05Z
          owner: notRoot
          group: notRoot

      # If `dst` ends with a `/`, it'll create the given path and copy the given
      # `src` into it, the same way `cp` works with and without trailing `/`.
      - src: ./foo/bar/*
        dst: /usr/local/myapp/

      # Using the type 'dir', empty directories can be created. When building
      # RPMs, however, this type has another important purpose: Claiming
      # ownership of that folder. This is important because when upgrading or
      # removing an RPM package, only the directories for which it has claimed
      # ownership are removed. However, you should not claim ownership of a
      # folder that is created by the OS or a dependency of your package.
      #
      # A directory in the build environment can optionally be provided in the
      # 'src' field in order copy mtime and mode from that directory without
      # having to specify it manually.
      - dst: /some/dir
        type: dir
        file_info:
          mode: 0700

    # Scripts to execute during the installation of the package. (overridable)
    #
    # Keys are the possible targets during the installation process
    # Values are the paths to the scripts which will be executed.
    scripts:
      preinstall: "scripts/preinstall.sh"
      postinstall: "scripts/postinstall.sh"
      preremove: "scripts/preremove.sh"
      postremove: "scripts/postremove.sh"

    # All fields above marked as `overridable` can be overridden for a given
    # package format in this section.
    overrides:
      # The dependencies override can for example be used to provide version
      # constraints for dependencies where  different package formats use
      # different versions or for dependencies that are named differently.
      deb:
        dependencies:
          - baz (>= 1.2.3-0)
          - some-lib-dev
        # ...
      rpm:
        dependencies:
          - baz >= 1.2.3-0
          - some-lib-devel
        # ...
      apk:
        # ...

    # Custom configuration applied only to the RPM packager.
    rpm:
      # RPM specific scripts.
      scripts:
        # The pretrans script runs before all RPM package transactions / stages.
        pretrans: ./scripts/pretrans.sh
        # The posttrans script runs after all RPM package transactions / stages.
        posttrans: ./scripts/posttrans.sh

      # The package summary.
      #
      # Default: first line of the description
      summary: Explicit Summary for Sample Package

      # The package group.
      # This option is deprecated by most distros but required by old distros
      # like CentOS 5 / EL 5 and earlier.
      group: Unspecified

      # The packager is used to identify the organization that actually packaged
      # the software, as opposed to the author of the software.
      # `maintainer` will be used as fallback if not specified.
      # This will expand any env var you set in the field, eg packager: ${PACKAGER}
      packager: GoReleaser <staff@goreleaser.com>

      # Compression algorithm (gzip (default), lzma or xz).
      compression: lzma

      # Prefixes for relocatable packages.
      #
      # Since: v1.20.
      prefixes:
        - /usr/bin

      # The package is signed if a key_file is set
      signature:
        # PGP secret key file path (can also be ASCII-armored).
        # The passphrase is taken from the environment variable
        # `$NFPM_ID_RPM_PASSPHRASE` with a fallback to `$NFPM_ID_PASSPHRASE`,
        # where ID is the id of the current nfpm config.
        # The id will be transformed to uppercase.
        # E.g. If your nfpm id is 'default' then the rpm-specific passphrase
        # should be set as `$NFPM_DEFAULT_RPM_PASSPHRASE`
        #
        # Templates: allowed
        key_file: "{{ .Env.GPG_KEY_PATH }}"

    # Custom configuration applied only to the Deb packager.
    deb:
      # Lintian overrides
      lintian_overrides:
        - statically-linked-binary
        - changelog-file-missing-in-native-package

      # Custom deb special files.
      scripts:
        # Deb rules script.
        rules: foo.sh
        # Deb templates file, when using debconf.
        templates: templates

      # Custom deb triggers
      triggers:
        # register interest on a trigger activated by another package
        # (also available: interest_await, interest_noawait)
        interest:
          - some-trigger-name
        # activate a trigger for another package
        # (also available: activate_await, activate_noawait)
        activate:
          - another-trigger-name

      # Packages which would break if this package would be installed.
      # The installation of this package is blocked if `some-package`
      # is already installed.
      breaks:
        - some-package

      # The package is signed if a key_file is set
      signature:
        # PGP secret key file path (can also be ASCII-armored).
        # The passphrase is taken from the environment variable
        # `$NFPM_ID_DEB_PASSPHRASE` with a fallback to `$NFPM_ID_PASSPHRASE`,
        # where ID is the id of the current nfpm config.
        # The id will be transformed to uppercase.
        # E.g. If your nfpm id is 'default' then the deb-specific passphrase
        # should be set as `$NFPM_DEFAULT_DEB_PASSPHRASE`
        #
        # Templates: allowed
        key_file: "{{ .Env.GPG_KEY_PATH }}"

        # The type describes the signers role, possible values are "origin",
        # "maint" and "archive".
        #
        # Default: 'origin'
        type: origin

    apk:
      # APK specific scripts.
      scripts:
        # The preupgrade script runs before APK upgrade.
        preupgrade: ./scripts/preupgrade.sh
        # The postupgrade script runs after APK.
        postupgrade: ./scripts/postupgrade.sh

      # The package is signed if a key_file is set
      signature:
        # PGP secret key file path (can also be ASCII-armored).
        # The passphrase is taken from the environment variable
        # `$NFPM_ID_APK_PASSPHRASE` with a fallback to `$NFPM_ID_PASSPHRASE`,
        # where ID is the id of the current nfpm config.
        # The id will be transformed to uppercase.
        # E.g. If your nfpm id is 'default' then the apk-specific passphrase
        # should be set as `$NFPM_DEFAULT_APK_PASSPHRASE`
        #
        # Templates: allowed
        key_file: "{{ .Env.GPG_KEY_PATH }}"

        # The name of the signing key. When verifying a package, the signature
        # is matched to the public key store in /etc/apk/keys/<key_name>.rsa.pub.
        #
        # Default: maintainer's email address
        # Templates: allowed (since v1.15)
        key_name: origin

    archlinux:
      # Archlinux-specific scripts
      scripts:
        # The preupgrade script runs before pacman upgrades the package.
        preupgrade: ./scripts/preupgrade.sh
        # The postupgrade script runs after pacman upgrades the package.
        postupgrade: ./scripts/postupgrade.sh

      # The pkgbase can be used to explicitly specify the name to be used to refer
      # to a group of packages. See: https://wiki.archlinux.org/title/PKGBUILD#pkgbase.
      pkgbase: foo

      # The packager refers to the organization packaging the software, not to be confused
      # with the maintainer, which is the person who maintains the software.
      packager: GoReleaser <staff@goreleaser.com>
```

Learn more about the [name template engine](https://goreleaser.com/customization/templates/).



## Checksums 校验

GoReleaser 会生成一个文件并将其与版本一起上传，以便您的用户可以验证下载的文件是否正确。

该部分允许自定义文件名：

```jsx
# .goreleaser.yaml
checksum:
  # You can change the name of the checksums file.
  #
  # Default: {{ .ProjectName }}_{{ .Version }}_checksums.txt
  # Templates: allowed
  name_template: "{{ .ProjectName }}_checksums.txt"

  # Algorithm to be used.
  # Accepted options are sha256, sha512, sha1, crc32, md5, sha224 and sha384.
  #
  # Default: sha256.
  algorithm: sha256

  # IDs of artifacts to include in the checksums file.
  #
  # If left empty, all published binaries, archives, linux packages and source archives
  # are included in the checksums file.
  ids:
    - foo
    - bar

  # Disable the generation/upload of the checksum file.
  disable: true

  # You can add extra pre-existing files to the checksums file.
  # The filename on the checksum will be the last part of the path (base).
  # If another file with the same name exists, the last one found will be used.
  #
  # Templates: allowed
  extra_files:
    - glob: ./path/to/file.txt
    - glob: ./glob/**/to/**/file/**/*
    - glob: ./glob/foo/to/bar/file/foobar/override_from_previous
    - glob: ./single_file.txt
      name_template: file.txt # note that this only works if glob matches 1 file only

  # Additional templated extra files to add to the checksum.
  # Those files will have their contents pass through the template engine,
  # and its results will be added to the checksum.
  #
  # Since: v1.17 (pro)
  # This feature is only available in GoReleaser Pro.
  # Templates: allowed
  templated_extra_files:
    - src: LICENSE.tpl
      dst: LICENSE.txt
```

## Snapcraft Packages (snaps) Snapcraft Packages

GoReleaser也可以生成软件包。Snaps 是一种新的打包格式，可让您将项目直接发布到 Ubuntu 商店。从那里，它可以安装在所有受支持的Linux发行版中，并进行自动和事务性更新。

您可以在 snapcraft 文档中阅读更多相关信息。

**Snaps是适用于桌面**、**云**和**物联网**的 Linux 应用程序包，易于安装、安全、跨平台且无依赖性。

它们会**自动更新，并且通常在有限的**基于**事务的**环境中运行。**安全性和稳健性**是其主要特点，此外还**易于安装**、**易于维护**和**易于升级**。

**Snapd 发布流程**

snapd 是管理和维护快照的后台服务。它本身可以作为 snap 包或传统的 Linux 软件包（例如 *deb* 或 RPM）提供。

有两种类型的发布；主要和次要版本，由其版本号的数字状态表示，并带有次要句点和为次要版本保留的数字：

- 主要版本发布：2.53、2.54、2.55
- 次要版本发布：2.53.1、2.53.2

主要版本和次要版本之间的区别在于其计划、准备和动机。每隔几周就有一个主要发布周期，但有时我们需要包含较小更改和修复的中间次要版本发布。

主要版本和后续次要版本（例如 2.53 -> 2.53.1）之间的差异尽可能小且有针对性，并省略主要代码重构和新功能。这并不总是可能的，因为有时错误或功能很复杂，但这是我们的首要目标。

**逐步发布流程**

- [https://gist.github.com/baymaxium/e1602202e7a3ef53a723ae14a3e928bc](https://gist.github.com/baymaxium/e1602202e7a3ef53a723ae14a3e928bc)

**使用Snapcraft构建发布Snap安装包**

生成一个初始工程：

```jsx
$ snapcraft init
Created snap/snapcraft.yaml.
```

## Docker Images

GoReleaser 可以构建和推送 Docker 镜像。让我们看看它是如何工作的。

您可以声明多个 Docker 映像。它们将与节生成的二进制文件和节生成的包进行匹配。

如果您只有一个设置，则配置就像将映像名称添加到文件中一样简单：

```jsx
dockers:
  - image_templates:
      - user/repo
```

您还需要在项目的根文件夹中创建一个 `Dockerfile`：

```jsx
FROM scratch
ENTRYPOINT ["/mybin"]
COPY mybin /
```

此配置将生成并推送名为 的 Docker 映像。

**Customization 定制**

```jsx
# .goreleaser.yaml
dockers:
  # You can have multiple Docker images.
  - #
    # ID of the image, needed if you want to filter by it later on (e.g. on custom publishers).
    id: myimg

    # GOOS of the built binaries/packages that should be used.
    # Default: 'linux'
    goos: linux

    # GOARCH of the built binaries/packages that should be used.
    # Default: 'amd64'
    goarch: amd64

    # GOARM of the built binaries/packages that should be used.
    # Default: '6'
    goarm: ""

    # GOAMD64 of the built binaries/packages that should be used.
    # Default: 'v1'
    goamd64: "v2"

    # IDs to filter the binaries/packages.
    ids:
      - mybuild
      - mynfpm

    # Templates of the Docker image names.
    #
    # Templates: allowed
    image_templates:
      - "myuser/myimage:latest"
      - "myuser/myimage:{{ .Tag }}"
      - "myuser/myimage:{{ .Tag }}-{{ .Env.FOOBAR }}"
      - "myuser/myimage:v{{ .Major }}"
      - "gcr.io/myuser/myimage:latest"

    # Skips the docker build.
    # Could be useful if you want to skip building the windows docker image on
    # linux, for example.
    #
    # Templates: allowed
    # Since: v1.14 (pro)
    # This option is only available on GoReleaser Pro.
    skip_build: false

    # Skips the docker push.
    # Could be useful if you also do draft releases.
    #
    # If set to auto, the release will not be pushed to the Docker repository
    #  in case there is an indicator of a prerelease in the tag, e.g. v1.0.0-rc1.
    #
    # Templates: allowed (since v1.19)
    skip_push: false

    # Path to the Dockerfile (from the project root).
    #
    # Default: 'Dockerfile'
    dockerfile: "{{ .Env.DOCKERFILE }}"

    # Set the "backend" for the Docker pipe.
    #
    # Valid options are: docker, buildx, podman.
    #
    # Podman is a GoReleaser Pro feature and is only available on Linux.
    #
    # Default: 'docker'
    use: docker

    # Docker build flags.
    #
    # Templates: allowed
    build_flag_templates:
      - "--pull"
      - "--label=org.opencontainers.image.created={{.Date}}"
      - "--label=org.opencontainers.image.title={{.ProjectName}}"
      - "--label=org.opencontainers.image.revision={{.FullCommit}}"
      - "--label=org.opencontainers.image.version={{.Version}}"
      - "--build-arg=FOO={{.Env.Bar}}"
      - "--platform=linux/arm64"

    # Extra flags to be passed down to the push command.
    push_flags:
      - --tls-verify=false

    # If your Dockerfile copies files other than binaries and packages,
    # you should list them here as well.
    # Note that GoReleaser will create the same structure inside a temporary
    # folder, so if you add `foo/bar.json` here, on your Dockerfile you can
    # `COPY foo/bar.json /whatever.json`.
    # Also note that the paths here are relative to the folder in which
    # GoReleaser is being run (usually the repository root folder).
    # This field does not support wildcards, you can add an entire folder here
    # and use wildcards when you `COPY`/`ADD` in your Dockerfile.
    extra_files:
      - config.yml

    # Additional templated extra files to add to the Docker image.
    # Those files will have their contents pass through the template engine,
    # and its results will be added to the build context the same way as the
    # extra_files field above.
    #
    # Since: v1.17 (pro)
    # This feature is only available in GoReleaser Pro.
    # Templates: allowed
    templated_extra_files:
      - src: LICENSE.tpl
        dst: LICENSE.txt
        mode: 0644
```

## Docker Images

GoReleaser 可以构建和推送 Docker 镜像。让我们看看它是如何工作的。

您可以声明多个 Docker 映像。它们将与节生成的二进制文件和节生成的包进行匹配。

如果您只有一个 build 设置，则配置就像将映像名称添加到文件中一样简单：

```jsx
dockers:
  - image_templates:
      - user/repo
```

您还需要在项目的根文件夹中创建一个：

```jsx
FROM scratch
ENTRYPOINT ["/mybin"]
COPY mybin /
```

此配置将生成并推送名为 的 Docker 映像。

**Customization**

```jsx
# .goreleaser.yaml
dockers:
  # You can have multiple Docker images.
  - #
    # ID of the image, needed if you want to filter by it later on (e.g. on custom publishers).
    id: myimg

    # GOOS of the built binaries/packages that should be used.
    # Default: 'linux'
    goos: linux

    # GOARCH of the built binaries/packages that should be used.
    # Default: 'amd64'
    goarch: amd64

    # GOARM of the built binaries/packages that should be used.
    # Default: '6'
    goarm: ""

    # GOAMD64 of the built binaries/packages that should be used.
    # Default: 'v1'
    goamd64: "v2"

    # IDs to filter the binaries/packages.
    ids:
      - mybuild
      - mynfpm

    # Templates of the Docker image names.
    #
    # Templates: allowed
    image_templates:
      - "myuser/myimage:latest"
      - "myuser/myimage:{{ .Tag }}"
      - "myuser/myimage:{{ .Tag }}-{{ .Env.FOOBAR }}"
      - "myuser/myimage:v{{ .Major }}"
      - "gcr.io/myuser/myimage:latest"

    # Skips the docker build.
    # Could be useful if you want to skip building the windows docker image on
    # linux, for example.
    #
    # Templates: allowed
    # Since: v1.14 (pro)
    # This option is only available on GoReleaser Pro.
    skip_build: false

    # Skips the docker push.
    # Could be useful if you also do draft releases.
    #
    # If set to auto, the release will not be pushed to the Docker repository
    #  in case there is an indicator of a prerelease in the tag, e.g. v1.0.0-rc1.
    #
    # Templates: allowed (since v1.19)
    skip_push: false

    # Path to the Dockerfile (from the project root).
    #
    # Default: 'Dockerfile'
    dockerfile: "{{ .Env.DOCKERFILE }}"

    # Set the "backend" for the Docker pipe.
    #
    # Valid options are: docker, buildx, podman.
    #
    # Podman is a GoReleaser Pro feature and is only available on Linux.
    #
    # Default: 'docker'
    use: docker

    # Docker build flags.
    #
    # Templates: allowed
    build_flag_templates:
      - "--pull"
      - "--label=org.opencontainers.image.created={{.Date}}"
      - "--label=org.opencontainers.image.title={{.ProjectName}}"
      - "--label=org.opencontainers.image.revision={{.FullCommit}}"
      - "--label=org.opencontainers.image.version={{.Version}}"
      - "--build-arg=FOO={{.Env.Bar}}"
      - "--platform=linux/arm64"

    # Extra flags to be passed down to the push command.
    push_flags:
      - --tls-verify=false

    # If your Dockerfile copies files other than binaries and packages,
    # you should list them here as well.
    # Note that GoReleaser will create the same structure inside a temporary
    # folder, so if you add `foo/bar.json` here, on your Dockerfile you can
    # `COPY foo/bar.json /whatever.json`.
    # Also note that the paths here are relative to the folder in which
    # GoReleaser is being run (usually the repository root folder).
    # This field does not support wildcards, you can add an entire folder here
    # and use wildcards when you `COPY`/`ADD` in your Dockerfile.
    extra_files:
      - config.yml

    # Additional templated extra files to add to the Docker image.
    # Those files will have their contents pass through the template engine,
    # and its results will be added to the build context the same way as the
    # extra_files field above.
    #
    # Since: v1.17 (pro)
    # This feature is only available in GoReleaser Pro.
    # Templates: allowed
    templated_extra_files:
      - src: LICENSE.tpl
        dst: LICENSE.txt
        mode: 0644
```

请注意，您必须手动登录到要推送到的Docker注册表 - GoReleaser不会自行登录。

> 请注意，您必须手动登录到要推送到的Docker注册表 - GoReleaser不会自行登录。

这些设置应该允许您生成多个 Docker 映像，例如，使用多个语句，以及为项目中的每个二进制文件生成一个映像或一个具有多个二进制文件的映像，以及安装生成的包而不是手动复制二进制文件和配置。

### 通用映像名称

某些用户可能希望使其映像名称尽可能通用。这可以通过在定义中添加模板语言来实现：

```jsx
# .goreleaser.yaml
project_name: foo
dockers:
  - image_templates:
      - "myuser/{{.ProjectName}}"
```

这将生成并发布以下映像：

- `myuser/foo`

### 保持当前主要内容的 docker 映像更新

一些用户可能想要推送 docker 标记 、 以及何时（例如）构建。这可以通过使用多个：

```jsx
# .goreleaser.yaml
dockers:
  - image_templates:
      - "myuser/myimage:{{ .Tag }}"
      - "myuser/myimage:v{{ .Major }}"
      - "myuser/myimage:v{{ .Major }}.{{ .Minor }}"
      - "myuser/myimage:latest"
```

这将生成并发布以下映像：

- `myuser/myimage:v1.6.4`
- `myuser/myimage:v1`
- `myuser/myimage:v1.6`
- `myuser/myimage:latest`

通过这些设置，您可以希望推送多个具有多个标签的 Docker 映像。

### 发布到多个 docker 注册表

某些用户可能希望将映像推送到多个 docker 注册表。这可以通过使用多个：

```jsx
# .goreleaser.yaml
dockers:
  - image_templates:
      - "docker.io/myuser/myimage:{{ .Tag }}"
      - "docker.io/myuser/myimage:latest"
      - "gcr.io/myuser/myimage:{{ .Tag }}"
      - "gcr.io/myuser/myimage:latest"
```

这会生成以下映像并将其发布到 和 ：

- `myuser/myimage:v1.6.4`
- `myuser/myimage:latest`
- `gcr.io/myuser/myimage:v1.6.4`
- `gcr.io/myuser/myimage:latest`

### 应用 Docker 构建标志

可以使用 应用生成标志。这些标志必须是有效的 Docker 构建标志。

```jsx
# .goreleaser.yaml
dockers:
  - image_templates:
      - "myuser/myimage"
    build_flag_templates:
      - "--pull"
      - "--label=org.opencontainers.image.created={{.Date}}"
      - "--label=org.opencontainers.image.title={{.ProjectName}}"
      - "--label=org.opencontainers.image.revision={{.FullCommit}}"
      - "--label=org.opencontainers.image.version={{.Version}}"
```

这将执行以下命令：

```jsx
docker build -t myuser/myimage . \
  --pull \
  --label=org.opencontainers.image.created=2020-01-19T15:58:07Z \
  --label=org.opencontainers.image.title=mybinary \
  --label=org.opencontainers.image.revision=da39a3ee5e6b4b0d3255bfef95601890afd80709 \
  --label=org.opencontainers.image.version=1.6.4
```

### 将特定的构建器与 Docker buildx 一起使用

如果启用，则在构建映像时使用上下文构建器。此构建器始终可用，并由 Docker 引擎中的 BuildKit 提供支持。如果要使用其他构建器，可以使用以下字段指定它：

```jsx
# .goreleaser.yaml
dockers:
  - image_templates:
      - "myuser/myimage"
    use: buildx
    build_flag_templates:
      - "--builder=mybuilder"
```

### Podman

您可以使用而不是通过在配置上设置为：

```jsx
# .goreleaser.yaml
dockers:
  - image_templates:
      - "myuser/myimage"
    use: podman
```

请注意，GoReleaser 不会为您安装 Podman，也不会更改其任何配置。

## Docker Manifests

GoReleaser 还可以使用该工具创建和推送 Docker 多平台映像。

无需切换设备，在 Apple M2 芯片的机器上我们可以直接构建 `amd64` 也就是 Linux 平台镜像，`docker build` 命令提供了 `--platform` 参数可以构建跨平台镜像。

```jsx
docker build --platform=linux/amd64 -t kubecub/echo-platform-amd64 .
```

运行不同平台的镜像会怎么样：

你也许会好奇，在 Apple M2 芯片的主机设备上运行 `amd64` 平台镜像会怎样。目前咱们构建的这个简单镜像其实是能够运行的，只不过会得到一条警告信息：

```jsx
$ docker run --rm kubecub/echo-platform-amd64
WARNING: The requested image's platform (linux/amd64) does not match the detected host platform (linux/arm64/v8) and no specific platform was requested
Linux buildkitsandbox 5.15.49-linuxkit #1 SMP PREEMPT Tue Sep 13 07:51:32 UTC 2022 x86_64 Linux
```

输出内容中的 `x86_64` 就表示 `AMD64` 架构。

> 注意：虽然这个简单的镜像能够运行成功，但如果容器内部程序不支持跨平台，`amd64` 平台镜像无法在 `arm64` 平台运行成功。

**使用 manifest 合并多平台镜像**

我们可以使用 `docker manifest` 的子命令 `create` 创建一个 `manifest list`，即将多个平台的镜像合并为一个镜像。

`create` 命令用法很简单，后面跟的第一个参数 `jianghushinian/echo-platform` 即为合并后的镜像，从第二个参数开始可以指定一个或多个不同平台的镜像。

```jsx
docker manifest create kubecub/echo-platform kubecub/echo-platform-arm64 kubecub/echo-platform-amd64
```

浏览器中登录 [Docker Hub](https://link.juejin.cn/?target=https%3A%2F%2Fhub.docker.com%2F) 查看推送成功的镜像：

> 进入镜像信息详情页面的 `Tags` 标签，能够看到镜像支持 `amd64`、`arm64/v8` 这两个平台。

### Manifest 命令

可以发现，`docker manifest` 共提供了 `annotate`、`create`、`inspect`、`push`、`rm` 这 5 个子命。

可以发现，`create` 子命令支持两个可选参数 `-a/--amend` 用来修订已存在的多架构镜像。

指定 `--insecure` 参数则允许使用不安全的（非 https）镜像仓库。

### push

`push` 子命令我们也见过了，使用 `push` 可以将多架构镜像推送到镜像仓库。

同样的，`push` 也有一个 `--insecure` 参数允许使用不安全的（非 https）镜像仓库。

- `p/--purge` 选项的作用是推送本地镜像到远程仓库后，删除本地 `manifest list`。

### inspect

`inspect` 用来查看 `manifest`/`manifest list` 所包含的镜像信息。

`--insecure` 参数允许使用不安全的（非 https）镜像仓库。这已经是我们第三次看见这个参数了，这也验证了 `docker manifest` 命令需要联网才能使用的说法，因为这些子命令基本都涉及到和远程镜像仓库的交互。

### annotate

`annotate` 子命令可以给一个本地镜像 `manifest` 添加附加的信息。这有点像 K8s Annotations 的意思。

可选参数列表如下：

| 选项          | 描述                                                         |
| ------------- | ------------------------------------------------------------ |
| --arch        | 设置 CPU 架构信息。                                          |
| --os          | 设置操作系统信息。                                           |
| --os-features | 设置操作系统功能信息。                                       |
| --os-version  | 设置操作系统版本信息。                                       |
| --variant     | 设置 CPU 架构的 variant 信息（翻译过来是“变种”的意思），如 ARM 架构的 v7、v8 等。 |

## rm

最后要介绍的子命令是 `rm`，使用 `rm` 可以删除本地一个或多个多架构镜像（`manifest lists`）。

### Customization

您可以在一次 GoReleaser 运行中创建多个清单，以下是所有可用的选项：

```jsx
# .goreleaser.yaml
docker_manifests:
  # You can have multiple Docker manifests.
-
  # ID of the manifest, needed if you want to filter by it later on (e.g. on
  # custom publishers).
  id: myimg

  # Name for the manifest.
  #
  # Templates: allowed
  name_template: "foo/bar:{{ .Version }}"

  # Image name to be added to this manifest.
  #
  # Templates: allowed
  image_templates:
  - "foo/bar:{{ .Version }}-amd64"
  - "foo/bar:{{ .Version }}-arm64v8"

  # Extra flags to be passed down to the manifest create command.
  create_flags:
  - --insecure

  # Extra flags to be passed down to the manifest push command.
  push_flags:
  - --insecure

  # Skips the Docker manifest.
  # If you set this to `false` or `auto` on your source Docker configuration,
  #  you'll probably want to do the same here.
  #
  # If set to `auto`, the manifest will not be created in case there is an
  #  indicator of a prerelease in the tag, e.g. v1.0.0-rc1.
  #
  # Templates: allowed (since v1.19)
  skip_push: false

  # Set the "backend" for the Docker manifest pipe.
  # Valid options are: docker, podman
  #
  # Relevant notes:
  # 1. podman is a GoReleaser Pro feature and is only available on Linux;
  # 2. if you set podman here, the respective docker configuration need to use
  #     podman too.
  #
  # Default: 'docker'
  use: docker
```

## KO

https://github.com/ko-build/ko

KO is Build and deploy Go applications

install docs:

[Installation - ko: Easy Go Containers](https://ko.build/install/)

User Actions:

```jsx
steps:
- uses: imjasonh/setup-ko@v0.6
```

**User Ko:** 

`ko`取决于 Docker 配置中配置的身份验证（通常`~/.docker/config.json`）。

✨**如果您可以使用 推送图片`docker push`，则您已经通过了身份验证`ko`！**✨

由于`ko`不需要`docker`，`ko login`还提供了一个使用用户名和密码登录容器映像注册表的界面，类似于`[docker login](https://docs.docker.com/engine/reference/commandline/login/)`.

此外，即使未在 Docker 配置中配置身份验证，也`ko`包含使用环境中配置的凭据对以下容器注册表进行身份验证的内置支持：

- Google 容器注册表和 Artifact 注册表，使用[应用程序默认凭据](https://cloud.google.com/docs/authentication/production)`gcloud`

  或

- Amazon Elastic Container Registry，使用[AWS 凭证](https://github.com/awslabs/amazon-ecr-credential-helper/#aws-credentials)

- Azure 容器注册表，使用[环境变量](https://github.com/chrismellard/docker-credential-acr-env/)

- GitHub Container Registry，使用`GITHUB_TOKEN`环境变量

`ko`取决于环境变量 ，`KO_DOCKER_REPO`来确定应将其构建的映像推送到何处。通常这将是远程注册表，例如：

- `KO_DOCKER_REPO=gcr.io/my-project`， 或者
- `KO_DOCKER_REPO=ghcr.io/my-org/my-repo`， 或者
- `KO_DOCKER_REPO=my-dockerhub-user`

**步骤：**

```jsx
echo "***" | docker login ghcr.io -u kuebcub --password-stdin
export GITHUB_TOKEN="******"
export KO_DOCKER_REPO=ghcr.io/kubecub/exporter; ko build ./cmd/exporter
```

**测试：**

```jsx
docker run -p 8080:8080 $(ko build ./cmd/app)
```

## Docker Images with Ko

请注意 ko 将再次构建您的二进制文件。这不应该过多地增加发布时间，因为它会在可能的情况下使用与构建管道相同的构建选项，因此结果可能会被缓存。

```jsx
# .goreleaser.yaml
kos:
-
  # ID of this image.
  id: foo

  # Build ID that should be used to import the build settings.
  build: build-id

  # Main path to build.
  #
  # Default: build.main
  main: ./cmd/...

  # Working directory used to build.
  #
  # Default: build.dir
  working_dir: .

  # Base image to publish to use.
  #
  # Default: 'cgr.dev/chainguard/static'
  base_image: alpine

  # Labels for the image.
  #
  # Since: v1.17
  labels:
    foo: bar

  # Repository to push to.
  #
  # Default: $KO_DOCKER_REPO
  repository: ghcr.io/foo/bar

  # Platforms to build and publish.
  #
  # Default: 'linux/amd64'
  platforms:
  - linux/amd64
  - linux/arm64

  # Tag to build and push.
  # Empty tags are ignored.
  #
  # Default: 'latest'
  # Templates: allowed
  tags:
  - latest
  - '{{.Tag}}'
  - '{{if not .Prerelease}}stable{{end}}'

  # Creation time given to the image
  # in seconds since the Unix epoch as a string.
  #
  # Since: v1.17
  # Templates: allowed
  creation_time: '{{.CommitTimestamp}}'

  # Creation time given to the files in the kodata directory
  # in seconds since the Unix epoch as a string.
  #
  # Since: v1.17
  # Templates: allowed
  ko_data_creation_time: '{{.CommitTimestamp}}'

  # SBOM format to use.
  #
  # Default: 'spdx'
  # Valid options are: spdx, cyclonedx, go.version-m and none.
  sbom: none

  # Ldflags to use on build.
  #
  # Default: build.ldflags
  ldflags:
  - foo
  - bar

  # Flags to use on build.
  #
  # Default: build.flags
  flags:
  - foo
  - bar

  # Env to use on build.
  #
  # Default: build.env
  env:
  - FOO=bar
  - SOMETHING=value

  # Bare uses a tag on the $KO_DOCKER_REPO without anything additional.
  bare: true

  # Whether to preserve the full import path after the repository name.
  preserve_import_paths: true

  # Whether to use the base path without the MD5 hash after the repository name.
  base_import_paths: true
```

这是一个最小的例子：

```jsx
# .goreleaser.yml
before:
  hooks:
    - go mod tidy

builds:
  - env: [ "CGO_ENABLED=1" ]
    binary: test
    goos:
    - darwin
    - linux
    goarch:
    - amd64
    - arch64

kos:
  - repository: ghcr.io/caarlos0/test-ko
    tags:
    - '{{.Version}}'
    - latest
    bare: true
    preserve_import_paths: false
    platforms:
    - linux/amd64
    - linux/arm64
```

这将为 、 和 构建二进制文件，以及 Linux 的 Docker 映像和清单。

## 包的大小

```jsx
# .goreleaser.yaml
# Whether to enable the size reporting or not.
report_sizes: true
```

## Metadata 元数据

GoReleaser 在完成运行之前会在文件夹中创建一些元数据文件。

```jsx
# .goreleaser.yaml
#
metadata:
  # Set the modified timestamp on the metadata files.
  #
  # Templates: allowed.
  mod_timestamp: "{{ .CommitTimestamp }}"
```

## 签名校验

GoReleaser 提供了对可执行文件和档案进行签名的方法。

签名与校验和文件结合使用，通常仅对校验和文件进行签名就足够了。

默认配置为使用以下命令为校验和文件创建独立签名[GnuPG](https://www.gnupg.org/)，以及您的默认密钥。要启用签名只需添加

```jsx
# .goreleaser.yaml
signs:
  - artifacts: checksum
```

要自定义签名管道，您可以使用以下选项：

```jsx
# .goreleaser.yaml
signs:
  -
    # ID of the sign config, must be unique.
    #
    # Default: 'default'
    id: foo

    # Name of the signature file.
    #
    # Default: '${artifact}.sig'
    # Templates: allowed
    signature: "${artifact}_sig"

    # Path to the signature command
    #
    # Default: 'gpg'
    cmd: gpg2

    # Command line arguments for the command
    #
    # to sign with a specific key use
    # args: ["-u", "<key id, fingerprint, email, ...>", "--output", "${signature}", "--detach-sign", "${artifact}"]
    #
    # Default: ["--output", "${signature}", "--detach-sign", "${artifact}"]
    # Templates: allowed
    args: ["--output", "${signature}", "${artifact}", "{{ .ProjectName }}"]

    # Which artifacts to sign
    #
    #   all:      all artifacts
    #   none:     no signing
    #   checksum: only checksum file(s)
    #   source:   source archive
    #   package:  linux packages (deb, rpm, apk)
    #   archive:  archives from archive pipe
    #   binary:   binaries if archiving format is set to binary
    #   sbom:     any Software Bill of Materials generated for other artifacts
    #
    # Default: 'none'
    artifacts: all

    # IDs of the artifacts to sign.
    #
    # If `artifacts` is checksum or source, this fields has no effect.
    ids:
      - foo
      - bar

    # Stdin data to be given to the signature command as stdin.
    #
    # Templates: allowed
    stdin: '{{ .Env.GPG_PASSWORD }}'

    # StdinFile file to be given to the signature command as stdin.
    stdin_file: ./.password

    # Sets a certificate that your signing command should write to.
    # You can later use `${certificate}` or `.Env.certificate` in the `args` section.
    # This is particularly useful for keyless signing (for instance, with cosign).
    # Note that this should be a name, not a path.
    certificate: '{{ trimsuffix .Env.artifact ".tar.gz" }}.pem'

    # List of environment variables that will be passed to the signing command
    # as well as the templates.
    env:
    - FOO=bar
    - HONK=honkhonk

    # By default, the stdout and stderr of the signing cmd are discarded unless
    # GoReleaser is running with `--debug` set.
    # You can set this to true if you want them to be displayed regardless.
    #
    # Since: v1.2
    output: true
```

### 可用的变量名称

这些环境变量可能在接受模板的字段中可用：

- `${artifact}`：将被签名的工件的路径
- `${artifactID}`：将被签名的工件的ID
- `${certificate}`：证书文件名（如果提供）
- `${signature}`: 签名文件名

假设你有一个`cosign.key`在存储库根目录和`COSIGN_PWD`环境变量设置，一个简单的使用示例如下：

```jsx
# .goreleaser.yaml
signs:
- cmd: cosign
  stdin: '{{ .Env.COSIGN_PWD }}'
  args:
  - "sign-blob"
  - "--key=cosign.key"
  - "--output-signature=${signature}"
  - "${artifact}"
  - "--yes" # needed on cosign 2.0.0+
  artifacts: all
```

然后，您的用户可以通过以下方式验证签名：

`cosign verify-blob -key cosign.pub -signature file.tar.gz.sig file.tar.gz`

## 对 Docker 映像和清单进行签名

使用 GoReleaser 也可以对 Docker 映像和清单进行签名。该管道是根据通用标志管道设计的，并考虑了共签名。

```jsx
# .goreleaser.yml
docker_signs:
  -
    # ID of the sign config, must be unique.
    # Only relevant if you want to produce some sort of signature file.
    #
    # Default: 'default'
    id: foo

    # Path to the signature command
    #
    # Default: 'cosign'
    cmd: cosign

    # Command line arguments for the command
    #
    # Templates: allowed
    # Default: ["sign", "--key=cosign.key", "${artifact}@${digest}", "--yes"]
    args:
    - "sign"
    - "--key=cosign.key"
    - "--upload=false"
    - "${artifact}"
    - "--yes" # needed on cosign 2.0.0+

    # Which artifacts to sign
    #
    #   all:       all artifacts
    #   none:      no signing
    #   images:    only docker images
    #   manifests: only docker manifests
    #
    # Default: 'none'
    artifacts: all

    # IDs of the artifacts to sign.
    ids:
      - foo
      - bar

    # Stdin data to be given to the signature command as stdin.
    #
    # Templates: allowed
    stdin: '{{ .Env.COSIGN_PWD }}'

    # StdinFile file to be given to the signature command as stdin.
    stdin_file: ./.password

    # List of environment variables that will be passed to the signing command as well as the templates.
    env:
    - FOO=bar
    - HONK=honkhonk

    # By default, the stdout and stderr of the signing cmd are discarded unless GoReleaser is running with `--debug` set.
    # You can set this to true if you want them to be displayed regardless.
    #
    # Since: v1.2
    output: true
```

这些环境变量可能在可模板化的字段中可用：

- `${artifact}`: 将要签名的项目的路径
- `${digest}`: 将签名的映像/清单的摘要
- `${artifactID}`: 将要签名的项目的 ID
- `${certificate}`: 证书文件名（如果提供）

## Release

GoReleaser 可以使用当前标签创建 GitHub/GitLab/Gitea 版本，上传所有工件，并根据自上一个标签以来的新提交生成更改日志。

让我们看看 GitHub 部分可以自定义的内容：

```jsx
# .goreleaser.yaml
release:
  # Repo in which the release will be created.
  # Default is extracted from the origin remote URL or empty if its private hosted.
  github:
    owner: user
    name: repo

  # IDs of the archives to use.
  # Empty means all IDs.
  #
  # Default: []
  ids:
    - foo
    - bar

  # If set to true, will not auto-publish the release.
  # Available only for GitHub and Gitea.
  #
  # Default: false
  draft: false

  # Whether to remove existing draft releases with the same name before creating
  # a new one.
  # Only effective if `draft` is set to true.
  # Available only for GitHub.
  #
  # Default: false
  # Since: v1.11
  replace_existing_draft: false

  # Useful if you want to delay the creation of the tag in the remote.
  # You can create the tag locally, but not push it, and run GoReleaser.
  # It'll then set the `target_commitish` portion of the GitHub release to the
  # value of this field.
  # Only works on GitHub.
  #
  # Default: ''
  # Since: v1.11
  # Templates: allowed
  target_commitish: "{{ .Commit }}"

  # This allows to change which tag GitHub will create.
  # Usually you'll use this together with `target_commitish`, or if you want to
  # publish a binary from a monorepo into a public repository somewhere, without
  # the tag prefix.
  #
  # Default: '{{ .PrefixedCurrentTag }}'
  # Since: v1.19 (pro)
  # Templates: allowed
  tag: "{{ .CurrentTag }}"

  # If set, will create a release discussion in the category specified.
  #
  # Warning: do not use categories in the 'Announcement' format.
  #  Check https://github.com/goreleaser/goreleaser/issues/2304 for more info.
  #
  # Default is empty.
  discussion_category_name: General

  # If set to auto, will mark the release as not ready for production
  # in case there is an indicator for this in the tag e.g. v1.0.0-rc1
  # If set to true, will mark the release as not ready for production.
  # Default is false.
  prerelease: auto

  # If set to false, will NOT mark the release as "latest".
  # This prevents it from being shown at the top of the release list,
  # and from being returned when calling https://api.github.com/repos/OWNER/REPO/releases/latest.
  #
  # Available only for GitHub.
  #
  # Default is true.
  # Since: v1.20.
  make_latest: true

  # What to do with the release notes in case there the release already exists.
  #
  # Valid options are:
  # - `keep-existing`: keep the existing notes
  # - `append`: append the current release notes to the existing notes
  # - `prepend`: prepend the current release notes to the existing notes
  # - `replace`: replace existing notes
  #
  # Default is `keep-existing`.
  mode: append

  # Header for the release body.
  #
  # Templates: allowed
  header: |
    ## Some title ({{ .Date }})

    Welcome to this new release!

  # Footer for the release body.
  #
  # Templates: allowed
  footer: |
    ## Thanks!

    Those were the changes on {{ .Tag }}!

  # You can change the name of the release.
  #
  # Default: '{{.Tag}}' ('{{.PrefixedTag}}' on Pro)
  # Templates: allowed
  name_template: "{{.ProjectName}}-v{{.Version}} {{.Env.USER}}"

  # You can disable this pipe in order to not create the release on any SCM.
  # Keep in mind that this might also break things that depend on the release
  # URL, for instance, homebrew taps.
  #
  # Templates: allowed (since v1.15)
  disable: true

  # Set this to true if you want to disable just the artifact upload to the SCM.
  # If this is true, GoReleaser will still create the release with the
  # changelog, but won't upload anything to it.
  #
  # Since: v1.11
  # Templates: allowed (since v1.15)
  skip_upload: true

  # You can add extra pre-existing files to the release.
  # The filename on the release will be the last part of the path (base).
  # If another file with the same name exists, the last one found will be used.
  #
  # Templates: allowed
  extra_files:
    - glob: ./path/to/file.txt
    - glob: ./glob/**/to/**/file/**/*
    - glob: ./glob/foo/to/bar/file/foobar/override_from_previous
    - glob: ./single_file.txt
      name_template: file.txt # note that this only works if glob matches 1 file only

  # Additional templated extra files to add to the release.
  # Those files will have their contents pass through the template engine,
  # and its results will be added to the release.
  #
  # Since: v1.17 (pro)
  # This feature is only available in GoReleaser Pro.
  # Templates: allowed
  templated_extra_files:
    - src: LICENSE.tpl
      dst: LICENSE.txt
```

## GPG 认证

GitHub 支持多种 GPG 关键算法。如果您尝试添加使用不受支持的算法生成的密钥，则可能会遇到错误。

### 检查现有 GPG 密钥

使用该`gpg --list-secret-keys --keyid-format=long`命令列出您拥有公钥和私钥的 GPG 密钥的长格式。签署提交或标签需要私钥。

```jsx
gpg --list-secret-keys --keyid-format=long
```

### 生成新的 GPG 密钥

通过 git 的参数校验。配置：

```jsx
git config --global gpg.program gpg2
```

**生成密钥对：**

```jsx
gpg --full-generate-key
```

**检查密钥对：**

```jsx
gpg --list-secret-keys --keyid-format=long
```

从 GPG 密钥列表中，复制您要使用的 GPG 密钥 ID 的完整形式。

1. 在此示例中，GPG 密钥 ID 为`3AA5C34371567BD2`：

   ```
   $ gpg --list-secret-keys --keyid-format=long
   /Users/hubot/.gnupg/secring.gpg
   ------------------------------------
   sec   4096R/3AA5C34371567BD2 2016-03-10 [expires: 2017-03-10]
   uid                          Hubot <hubot@example.com>
   ssb   4096R/4BB6D45482678BE3 2016-03-10
   ```

2. 粘贴下面的文本，并将其替换为您要使用的 GPG 密钥 ID。在此示例中，GPG 密钥 ID 为`3AA5C34371567BD2`：

   ```
   gpg --armor --export 3AA5C34371567BD2
   # Prints the GPG key ID, in ASCII armor format
   ```

3. 复制您的 GPG 密钥，以 开头`----BEGIN PGP PUBLIC KEY BLOCK-----`和结尾`----END PGP PUBLIC KEY BLOCK-----`。

```jsx
cat /root/.gnupg/openpgp-revocs.d/4DDA37AE0F3AEA3044B33F1B1BAD6F395338EFDE.rev
```

然后就是将这个密钥链接到你的 GitHub 账户了。这个操作很简单，不介绍了

**告诉 Git 你的签名密钥：**

你还需要告诉 Git 关于你的 签名 密钥，因为 如果您有多个 GPG 密钥，则需要告诉 Git 使用哪一个。

1. 如果您之前已将 Git 配置为在使用 进行签名时使用不同的密钥格式，请取消设置此配置，以便使用`-gpg-sign`默认格式。`openpgp`

   ```
   git config --global --unset gpg.format
   ```

使用该`gpg --list-secret-keys --keyid-format=long`命令列出您拥有公钥和私钥的 GPG 密钥的长格式。签署提交或标签需要私钥。

```jsx
gpg --list-secret-keys --keyid-format=long
```

从 GPG 密钥列表中，复制您要使用的 GPG 密钥 ID 的完整形式。在此示例中，GPG 密钥 ID 为`3AA5C34371567BD2`：

```jsx
$ gpg --list-secret-keys --keyid-format=long
/Users/hubot/.gnupg/secring.gpg
------------------------------------
sec   4096R/3AA5C34371567BD2 2016-03-10 [expires: 2017-03-10]
uid                          Hubot <hubot@example.com>
ssb   4096R/4BB6D45482678BE3 2016-03-10
```

要在 Git 中设置主 GPG 签名密钥，请粘贴下面的文本，并替换为您要使用的 GPG 主密钥 ID。在此示例中，GPG 密钥 ID 为`3AA5C34371567BD2`：

```jsx
git config --global user.signingkey 3AA5C34371567BD2
```

或者，在设置子项时包含后缀`!`。在此示例中，GPG 子密钥 ID 为`4BB6D45482678BE3`：

```jsx
git config --global user.signingkey 4BB6D45482678BE3!
```

或者，要将 Git 配置为默认签署所有提交，请输入以下命令：

```jsx
git config --global commit.gpgsign true
```

+ [告诉 Git 你的 SSH 密钥](https://docs.github.com/en/authentication/managing-commit-signature-verification/telling-git-about-your-signing-key#telling-git-about-your-ssh-key)

您可以使用现有的 SSH 密钥来签署提交和标签，或生成专门用于签名的新密钥。有关更多信息，请参阅“[生成新的 SSH 密钥并将其添加到 ssh-agent](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent) ”。

**注意：**

我们可能需要将 `export GPG_TTY=$(tty)` 添加到环境变量中

### 签名标签

```jsx
$ git tag -s MYTAG
# Creates a signed tag
```

通过运行验证您的签名标签`git tag -v [tag-name]`。

```
$ git tag -v MYTAG
# Verifies the signed tag
```

## 云存储服务

允许您将工件上传到 Amazon S3、Azure Blob 和 Google GCS。

其实不仅仅是这些，还有包括国内的 COS 和 CSS

```jsx
# .goreleaser.yaml
blobs:
  # You can have multiple blob configs
  - # Cloud provider name:
    # - s3 for AWS S3 Storage
    # - azblob for Azure Blob Storage
    # - gs for Google Cloud Storage
    #
    # Templates: allowed
    provider: azblob

    # Set a custom endpoint, useful if you're using a minio backend or
    # other s3-compatible backends.
    #
    # Implies s3ForcePathStyle and requires provider to be `s3`
    #
    # Templates: allowed
    endpoint: https://minio.foo.bar

    # Sets the bucket region.
    # Requires provider to be `s3`
    #
    # Templates: allowed
    region: us-west-1

    # Disables SSL
    # Requires provider to be `s3`
    disableSSL: true

    # Bucket name.
    #
    # Templates: allowed
    bucket: goreleaser-bucket

    # IDs of the artifacts you want to upload.
    ids:
      - foo
      - bar

    # Path/name inside the bucket.
    #
    # Default: '{{ .ProjectName }}/{{ .Tag }}'
    # Templates: allowed
    folder: "foo/bar/{{.Version}}"

    # Whether to disable this particular upload configuration.
    #
    # Since: v1.17
    # Templates: allowed
    disable: '{{ neq .BLOB_UPLOAD_ONLY "foo" }}'

    # You can add extra pre-existing files to the bucket.
    # The filename on the release will be the last part of the path (base).
    # If another file with the same name exists, the last one found will be used.
    # These globs can also include templates.
    extra_files:
      - glob: ./path/to/file.txt
      - glob: ./glob/**/to/**/file/**/*
      - glob: ./glob/foo/to/bar/file/foobar/override_from_previous
      - glob: ./single_file.txt
        # Templates: allowed
        name_template: file.txt # note that this only works if glob matches 1 file only

    # Additional templated extra files to uploaded.
    # Those files will have their contents pass through the template engine,
    # and its results will be uploaded.
    #
    # Since: v1.17 (pro)
    # This feature is only available in GoReleaser Pro.
    # Templates: allowed
    templated_extra_files:
      - src: LICENSE.tpl
        dst: LICENSE.txt

  - provider: gs
    bucket: goreleaser-bucket
    folder: "foo/bar/{{.Version}}"
  - provider: s3
    bucket: goreleaser-bucket
    folder: "foo/bar/{{.Version}}"
```

### Fury.io (apt 和 rpm 存储库）

**这是一个高级功能**，但是 sealos 也使用了，用的是 bash 逻辑

您可以使用 GoReleaser 轻松地在 fury.io 上创建和存储库。

首先，您需要在 fury.io 上创建一个帐户并获取推送令牌。

然后，您需要将您的帐户名称传递给 GoReleaser，并将您的推送令牌作为名为 `FURY_TOKEN` ：

```jsx
# .goreleaser.yaml
furies:
- account: myaccount
```

这将自动上传您的所有文件。

**自定义：**

```jsx
# goreleaser.yaml

furies:
  -
    # fury.io account.
    # Config is skipped if empty
    account: "{{ .Env.FURY_ACCOUNT }}"

    # Skip the announcing feature in some conditions, for instance, when
    # publishing patch releases.
    # Any value different of 'true' will be considered 'false'.
    #
    # Templates: allowed
    skip: "{{gt .Patch 0}}"

    # Environment variable name to get the push token from.
    # You might want to change it if you have multiple fury configurations for
    # some reason.
    #
    # Default: 'FURY_TOKEN'
    secret_name: MY_ACCOUNT_FURY_TOKEN

    # IDs to filter by.
    # configurations get uploaded.
    ids:
      - packages

    # Formats to upload.
    # Available options are `deb` and `rpm`.
    #
    # Default: ['deb', 'rpm']
    formats:
      - deb
```

## Homebrew Taps

发布到 GitHub、GitLab 或 Gitea 后，GoReleaser 可以生成 *homebrew-tap* 并将其发布到您有权访问的存储库中。

## Announce

GoReleaser还可以在社交网络，聊天室和电子邮件上宣布新版本！

它在管道的最末端运行，可以使用命令的标志或通过 skip 属性跳过：

```jsx
# .goreleaser.yaml
announce:
  # Skip the announcing feature in some conditions, for instance, when
  # publishing patch releases.
  #
  # Any value different from 'true' is evaluated to false.
  #
  # Templates: allowed
  skip: "{{gt .Patch 0}}"
```

### 目前支持很多个账户

**Discode:** 

要使用 Discord，您需要创建一个 [Webhook](https://support.discord.com/hc/en-us/articles/228383668-Intro-to-Webhooks)，并在管道上设置以下环境变量：

- `DISCORD_WEBHOOK_ID`
- `DISCORD_WEBHOOK_TOKEN`

在此之后，您可以将以下部分添加到您的配置中：

```jsx
# .goreleaser.yaml
announce:
  discord:
    # Whether its enabled or not.
    enabled: true

    # Message template to use while publishing.
    #
    # Default: '{{ .ProjectName }} {{ .Tag }} is out! Check it out at {{ .ReleaseURL }}'
    # Templates: allowed
    message_template: 'Awesome project {{.Tag}} is out!'

    # Set author of the embed.
    #
    # Default: 'GoReleaser'
    author: ''

    # Color code of the embed. You have to use decimal numeral system, not hexadecimal.
    #
    # Default: '3888754' (the grey-ish from GoReleaser)
    color: ''

    # URL to an image to use as the icon for the embed.
    #
    # Default: 'https://goreleaser.com/static/avatar.png'
    icon_url: ''
```

要使其正常工作，您需要在管道上设置一些环境变量：

- `LINKEDIN_ACCESS_TOKEN`

> We currently don't support posting in groups.

然后，您可以在配置中添加类似以下内容的内容：

```jsx
# .goreleaser.yaml
announce:
  linkedin:
    # Whether its enabled or not.
    enabled: true

    # Message to use while publishing.
    #
    # Default: '{{ .ProjectName }} {{ .Tag }} is out! Check it out at {{ .ReleaseURL }}'
    message_template: 'Awesome project {{.Tag}} is out!'
```

### slack

和 discode 一样， slack 同样也是需要传入一个新的 `Webhook`

- `SLACK_WEBHOOK`

然后，您可以在配置中添加类似以下内容的内容：

```jsx
# .goreleaser.yaml
announce:
  slack:
    # Whether its enabled or not.
    enabled: true

    ****# Message template to use while publishing.
    #
    # Default: '{{ .ProjectName }} {{ .Tag }} is out! Check it out at {{ .ReleaseURL }}'
    # Templates: allowed
    message_template: 'Awesome project {{.Tag}} is out!'

    # The name of the channel that the user selected as a destination for webhook messages.
    channel: '#channel'

    # Set your Webhook's user name.
    username: ''

    # Emoji to use as the icon for this message. Overrides icon_url.
    icon_emoji: ''

    # URL to an image to use as the icon for this message.
    icon_url: ''

    # Blocks for advanced formatting, see: https://api.slack.com/messaging/webhooks#advanced_message_formatting
    # and https://api.slack.com/messaging/composing/layouts#adding-blocks.
    #
    # Attention: goreleaser doesn't check the full structure of the Slack API: please make sure that
    # your configuration for advanced message formatting abides by this API.
    #
    # Templates: allowed
    blocks: []

    # Attachments, see: https://api.slack.com/reference/messaging/attachments
    #
    # Attention: goreleaser doesn't check the full structure of the Slack API: please make sure that
    # your configuration for advanced message formatting abides by this API.
    #
    # Templates: allowed
    attachments: []
```

## 链接

- [https://docs.docker.com/engine/reference/commandline/manifest/](https://docs.docker.com/engine/reference/commandline/manifest/)
