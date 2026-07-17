# unity-cli-actions

GitHub Actions composite actions for running Unity Editor in CI using the
official [Unity CLI](https://docs.unity.com/en-us/unity-cli), without Docker.

## Actions

### `setup-unity-cli`

Installs the Unity CLI, installs the requested Unity Editor version, and
locates the resulting Unity executable.

```yaml
- name: Setup Unity CLI
  id: unity
  uses: yamachu/unity-cli-actions/setup-unity-cli@v1
  with:
    unity-version: "2022.3.22f1" # optional, default: lts
    cli-version: "" # optional, default: latest from manifest
    cli-channel: "beta" # optional, default: beta. alpha, beta, or empty for stable
```

Output: `editor-path` — absolute path to the installed Unity executable.

> [!NOTE]
> The Unity CLI is still under development, and the stable channel's manifest
> is sometimes unavailable — only the beta channel reliably has one. That's
> why `cli-channel` currently defaults to `beta`. This default will change
> once the stable channel is consistently available.

Runs on Linux, macOS, and Windows runners (Windows uses Unity's `install.ps1`
instead of `install.sh`).

> [!WARNING]
> This action installs the Unity CLI by piping Unity's installer script
> straight from their CDN (`curl ... | bash` on Linux/macOS, `irm ... | iex`-
> style on Windows). Pinning `cli-version` controls which Unity CLI build gets
> installed, but not the installer script itself — if Unity changes the
> installer's behavior, this action's behavior can change without a version
> bump on our end.

### `activate-unity-license`

Activates a Unity Personal license for use in CI.

On **Linux/Windows** this activates live against Unity's servers, following
[game-ci/unity-builder](https://github.com/game-ci/unity-builder)'s
Editor-flag approach (`-serial -username -password`). Requires a real human
Unity ID (service accounts have no Personal-license entitlement) and the
Personal-tier serial extracted from an existing `.ulf`.

See: https://game.ci/docs/gitlab/activation/#2-extracting-the-serial-from-a-personal-license

```yaml
- name: Activate Unity Personal license
  uses: yamachu/unity-cli-actions/activate-unity-license@v1
  with:
    editor-path: ${{ steps.unity.outputs.editor-path }}
    serial: ${{ secrets.UNITY_SERIAL }}
    username: ${{ secrets.UNITY_EMAIL }}
    password: ${{ secrets.UNITY_PASSWORD }}
```

On **macOS**, live serial activation over the Unity Licensing Client's IPC
channel is unreliable in CI — the entitlement license activates fine, but the
subsequent ULF activation request reliably times out after 30s. Instead,
provide a pre-activated `.ulf` license file (base64-encoded) via
`license-ulf`, which is decoded and placed directly at
`/Library/Application Support/Unity/Unity_lic.ulf`, skipping the live
activation round-trip entirely:

```yaml
- name: Activate Unity Personal license
  uses: yamachu/unity-cli-actions/activate-unity-license@v1
  with:
    editor-path: ${{ steps.unity.outputs.editor-path }}
    license-ulf: ${{ secrets.UNITY_LICENSE_ULF }}
```

To obtain the `.ulf` once: activate Unity manually on any machine (e.g. via
`unity -createManualActivationFile`, submit the resulting `.alf` at
https://license.unity3d.com, and download the returned `.ulf`), then
`base64 -i Unity_lic.ulf | pbcopy` and store it as the `UNITY_LICENSE_ULF`
secret.

## Full example

```yaml
jobs:
  test-with-unity:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7

      - name: Setup Unity CLI
        id: unity
        uses: yamachu/unity-cli-actions/setup-unity-cli@v1
        with:
          unity-version: "2022.3.22f1"

      - name: Activate Unity Personal license
        uses: yamachu/unity-cli-actions/activate-unity-license@v1
        with:
          editor-path: ${{ steps.unity.outputs.editor-path }}
          serial: ${{ secrets.UNITY_SERIAL }}
          username: ${{ secrets.UNITY_EMAIL }}
          password: ${{ secrets.UNITY_PASSWORD }}

      - name: Run tests
        env:
          UNITY_PATH: ${{ steps.unity.outputs.editor-path }}
        run: dotnet test tests/YourProject.Tests --configuration Release
```

## License

[MIT](LICENSE)

`activate-unity-license` reimplements steps from
[game-ci/unity-builder](https://github.com/game-ci/unity-builder) (also MIT).
See [THIRD_PARTY_LICENSES.md](THIRD_PARTY_LICENSES.md).
