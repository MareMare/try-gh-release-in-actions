# try-gh-release-in-actions
🧪📝 GitHub Actions で `gh release` を試してみます。

* [Using GitHub CLI in workflows](https://docs.github.com/en/actions/using-workflows/using-github-cli-in-workflows)
* [Assigning permissions to jobs](https://docs.github.com/en/actions/using-jobs/assigning-permissions-to-jobs)
* [`gh release create`](https://cli.github.com/manual/gh_release_create)
* [`gh release upload`](https://cli.github.com/manual/gh_release_upload)

## 💡 `gh: To use GitHub CLI in a GitHub Actions workflow, set the GH_TOKEN environment variable.`
```yml
  env:
    GH_TOKEN: ${{ github.token }}
```

## 💡 `HTTP 403: Resource not accessible by integration`
```ps1
Run gh release create v0.0.1-beta.4 --generate-notes --draft --prerelease
HTTP 403: Resource not accessible by integration (https://api.github.com/repos/MareMare/try-gh-release-in-actions/releases)
Error: Process completed with exit code 1.
```

👇

`Settings > Actions > General > Workflow permissions`

![](<doc/Resource not accessible by integration.png>)

## `gh release upload` でリリースアセットにアップロードする方法

アップロードするアセットは以下に格納されていることを前提とします。
```sh
.
└─artifacts
        Sandbox.0.0.0-beta.nupkg
        Sandbox.0.0.0-beta.snupkg
```

### `$(ls **/*.*nupkg)`
> [!NOTE]
> `bash` and/or `pwsh`

```yml
        run: gh release upload ${{ env.TAG_NAME }} $(ls **/*.*nupkg)
```

### `$(find artifacts -type f)`
> [!NOTE]
> `bash` only

```yml
        run: gh release upload ${{ env.TAG_NAME }} $(find artifacts -type f)
        shell: bash
```

### `find artifacts -type f | xargs`
> [!NOTE]
> `bash` only

```yml
        run: find artifacts -type f | xargs gh release upload ${{ env.TAG_NAME }}
        shell: bash
```

<details><summary>おためしメモ：</summary>

* ローカルの `pwsh` でおためし
```ps1
$TAG_NAME="v0.0.1-beta.1"
git tag $TAG_NAME
git push --tags

# 📦 Pack
dotnet pack --output artifacts

# 📝 Create Release $TAG_NAME
gh release create $TAG_NAME --generate-notes --draft --prerelease
# ➕ Upload assets
gh release upload $TAG_NAME $(ls **/*.*nupkg)
```

* ローカルの `bash` でおためし
```sh
export TAG_NAME="v0.0.1-beta.2"
git tag $TAG_NAME
git push --tags

# 📦 Pack
dotnet pack --output artifacts

# 📝 Create Release $TAG_NAME
gh release create $TAG_NAME --generate-notes --draft --prerelease
# ➕ Upload assets
gh release upload $TAG_NAME $(ls **/*.*nupkg)
```

* GitHub Actions でおためし (`bash`)
```yml
name: Sandbox
on:
  workflow_dispatch:
    inputs:
      tag:
        description: 'Tag name. eg) "v0.0.1, v0.0.1-beta.1"'
        default: 'v0.0.1-beta.1'
        required: false

env:
  TAG_NAME: ${{ inputs.tag || github.ref_name }}
  CONFIGURATION: Release
  DOTNET_VERSION: "8.0.x"

concurrency:
  group: ${{ github.workflow }}-${{ github.event.pull_request.number || github.ref }}
  cancel-in-progress: true

jobs:
  sandbox:
    name: 🍻 おためし
    runs-on: ubuntu-latest
    if: startsWith(inputs.tag || github.ref_name, 'v')
    steps:
      - name: 🛒 Checkout
        uses: actions/checkout@v4.1.1
        with:
          fetch-depth: 0
          filter: tree:0
      - name: ✨ Setup .NET ${{ env.DOTNET_VERSION }}
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: ${{ env.DOTNET_VERSION }}
      - name: 🛠️ Build
        run: dotnet build --configuration ${{ env.CONFIGURATION }}
      - name: 🧪 Test
        run: dotnet test --configuration ${{ env.CONFIGURATION }} --no-build --verbosity normal --collect:"XPlat Code Coverage" --logger trx --results-directory coverage  --filter 'Category!=local'

      - name: 📦 Pack
        run: dotnet pack --output ./artifacts
      - name: 📝 Create Release ${{ env.TAG_NAME }}
        run: gh release create ${{ env.TAG_NAME }} --generate-notes --draft --prerelease
      - name: ➕ Upload assets
        run: gh release upload ${{ env.TAG_NAME }} $(ls **/*.*nupkg)
        shell: bash
      - run: echo "### ✅ Release succeeded! ${{ env.TAG_NAME }} 🌟" >> $GITHUB_STEP_SUMMARY
```

</details>


## 参考
* [gh release create wildcard '*' not working on windows](https://github.com/cli/cli/issues/5099)
* [cli/cli/.github/workflows/deployment.yml](https://github.com/cli/cli/blob/541ce0e5b49269b8b39707b3d16cfbd01d79b9a0/.github/workflows/deployment.yml#L344C43-L344C49)
