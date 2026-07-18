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
> If you need an **Intel macOS** runner (`os: osx-x64` / GitHub's `*-intel`
> labels), use `macos-15-intel`, not `macos-latest` or `macos-26-intel`. On
> `macos-26-intel` the Editor's very first asset import hangs forever right
> after kicking off every built-in module import — no completion ever
> follows. It reproduces even with a bare empty project, isn't FMOD/audio
> related (no FMOD errors appear, and disabling audio via
> `ProjectSettings/AudioManager.asset`'s `m_DisableAudio: 1` makes no
> difference), and looks structurally like the same class of Editor↔helper-
> process IPC issue as the ULF activation hang described below, just hitting
> the asset-import-worker path instead of license activation.

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

On **macOS**, that same Editor-IPC flow reliably times out waiting for
`ULFActivationResponse` — a known, unresolved issue with the Editor's legacy
ULF activation round-trip on ephemeral macOS runners (see
[game-ci/unity-builder#690](https://github.com/game-ci/unity-builder/issues/690),
[#572](https://github.com/game-ci/unity-builder/issues/572)). Placing a
pre-activated `.ulf` file directly doesn't work around it either — `.ulf`
files are bound to the activating machine's hardware ID, which never matches
a fresh ephemeral runner ("Machine bindings don't match").

Instead, on macOS this action activates via Unity's standalone
`Unity.Licensing.Client` binary directly (bundled inside the installed
Editor's `.app`), bypassing the Editor's IPC path entirely — the same
approach used by
[RageAgainstThePixel/unity-cli](https://github.com/RageAgainstThePixel/unity-cli)
and [buildalon/activate-unity-license](https://github.com/buildalon/activate-unity-license),
which have confirmed-green CI activating Personal licenses on macOS-hosted
runners. Concretely, it runs `--activate-all --username --password
--include-personal` (per the client's own `--help`, `--serial` combined with
`--activate-ulf` is for PRO licenses only, so it's not used here). The
`with:` inputs are the same as above, but `serial` is unused on macOS — only
`editor-path`, `username`, and `password` matter there.

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
