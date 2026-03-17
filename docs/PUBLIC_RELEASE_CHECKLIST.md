# Release Checklist

## Goal

Ship a GitHub release that works cleanly for HACS custom-repository installs.

## Repository prerequisites

1. Keep the integration under `custom_components/vivosun_growhub/`
2. Keep a root `README.md`
3. Keep a root `hacs.json`
4. Keep a valid `manifest.json`
5. Ensure the repository is public and GitHub Actions are enabled

## Metadata updates

1. Update [manifest.json](../custom_components/vivosun_growhub/manifest.json)
   - `version`
   - `documentation`
   - `issue_tracker`
   - `codeowners` if needed
2. Update [pyproject.toml](../pyproject.toml) version to match
3. Verify [hacs.json](../hacs.json)
4. Refresh user-facing docs if behavior or supported devices changed

## Validation

1. Pass the normal CI workflow
2. Pass the HACS validation workflow
3. Pass the hassfest workflow
4. Run the local test suite
5. Perform one manual Home Assistant install smoke test
6. Perform one HACS custom repository install or upgrade smoke test

## HACS expectations

Current HACS usage in this repository relies on release assets:

- `hacs.json` sets `zip_release: true`
- the release asset must be named `vivosun_growhub.zip`
- the asset contents must match `custom_components/vivosun_growhub/`

## Release process

1. Merge the intended changes onto `main`
2. Bump the integration version
3. Commit the version bump
4. Create and push a git tag matching the version
5. Create a GitHub release for that tag
6. Verify the `Release` workflow uploads `vivosun_growhub.zip`
7. Verify the release page shows the asset and tag you expect

## Optional follow-up

1. Add the repository to HACS as a custom repository and test install from that release
2. After battle testing, decide whether to keep it as custom-repo-only or submit it to the default HACS list
3. If targeting the default HACS list later, make sure the domain is registered with `brands.home-assistant.io`

## Branding

Use neutral README branding.
Vendor brand assets can stay inside the integration for identification, but the project should not read like an official vendor release.

## References

- HACS publish docs: https://www.hacs.xyz/docs/publish/integration/
- Home Assistant integration file structure: https://developers.home-assistant.io/docs/creating_integration_file_structure/
- Home Assistant custom integration brands: https://developers.home-assistant.io/blog/2026/02/24/brands-proxy-api/
