# release-tools

Shared GitHub Actions for building, signing and publishing the releases of my projects, so every project releases
the same way. Each project keeps its own build steps and calls these for the common parts.

| Action | What it does |
|---|---|
| `actions/changelog-notes` | One version's section of a [Keep a Changelog](https://keepachangelog.com) `CHANGELOG.md`, as a file for the release notes |
| `actions/fetch-deps` | Copies a project's folder from the private `build-deps` repository (third-party files that must not be in a public repository); without a token it warns and copies nothing |
| `actions/sign` | Authenticode-signs Windows executables and installers. `method: none` signs nothing and warns; `certum-simplysign` follows once the certificate exists |
| `actions/publish-release` | Creates or updates the GitHub Release of a tag: the files, a `SHA256SUMS.txt`, the notes |

## Using it

```yaml
    steps:
      - uses: HrMarcussen/release-tools/actions/fetch-deps@v1
        with:
          token: ${{ secrets.BUILD_DEPS_TOKEN }}
          folder: MyProject
          destination: lib
      # ... the project's own build ...
      - uses: HrMarcussen/release-tools/actions/sign@v1
        with:
          files: dist/MyProject.exe
          method: ${{ vars.SIGN_METHOD || 'none' }}
          certum-user: ${{ secrets.CERTUM_USER }}
          certum-otp-secret: ${{ secrets.CERTUM_OTP_SECRET }}
      - id: notes
        uses: HrMarcussen/release-tools/actions/changelog-notes@v1
        with:
          version: ${{ github.ref_name }}
      - uses: HrMarcussen/release-tools/actions/publish-release@v1
        with:
          tag: ${{ github.ref_name }}
          files: dist/*.exe
          notes-file: ${{ steps.notes.outputs.file }}
```

A complete example is GlassLink's `.github/workflows/release.yml`.

## Setting up a project

1. **build-deps token** (only if the project needs files from `build-deps`): GitHub, Settings, Developer settings,
   Fine-grained tokens, Generate new token. Resource owner: me; Repository access: only `build-deps`; Permissions:
   Contents read-only; an expiry date. Store it in the project as the **environment** secret `BUILD_DEPS_TOKEN` of the
   `release` environment (Settings, Environments), so only the release job can read it, not ordinary CI runs.
2. **Environment `release`** in the project (Settings, Environments): the release job runs in it. Give it a required
   reviewer (me), so nothing is signed or published without an approval, and put the signing secrets there, not in
   the repository's secrets.
3. **Signing** (once the certificate exists): environment secrets `CERTUM_USER` and `CERTUM_OTP_SECRET`, repository
   variable `SIGN_METHOD` = `certum-simplysign`.

## Security

- The signing secrets exist only in each project's protected `release` environment; a release job waits for an
  approval before it can read them.
- Only official software is meant to handle them: Certum's SimplySign Desktop and Microsoft's signtool, driven by
  the few lines in `actions/sign`. No third-party signing service.
- Third-party files come only from the private `build-deps` repository, through a read-only token for that one
  repository, and their SHA-256 is printed in the log.
- Projects pin these actions to a tag (`@v1`) or a commit; changes to this repository are reviewed like code.

## Licence

GNU General Public License version 3 or later ([LICENSE](LICENSE)).
