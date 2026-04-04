# Zyro ISO

A custom fork of [omarchy-iso](https://github.com/basecamp/omarchy-iso). Builds a bootable Arch Linux ISO that automates installation using the [Omarchy Installer](https://github.com/basecamp/omarchy) as the base.

## Creating the ISO

Run `./bin/zyro-iso-make` and the output goes into `./release`. You can build from your local $ZYRO_PATH for testing by using `--local-source` or from a checkout of the dev branch (instead of master) by using `--dev`.

### Environment Variables

You can customize the repositories used during the build process by passing in variables:

- `ZYRO_INSTALLER_REPO` - GitHub repository for the installer (default: `basecamp/omarchy`)
- `ZYRO_INSTALLER_REF` - Git ref (branch/tag) for the installer (default: `master`)

Example usage:
```bash
ZYRO_INSTALLER_REPO="myuser/omarchy-fork" ZYRO_INSTALLER_REF="some-feature" ./bin/zyro-iso-make
```

## Testing the ISO

Run `./bin/zyro-iso-boot [release/zyro.iso]`.

## Signing the ISO

Run `./bin/zyro-iso-sign [gpg-user] [release/zyro.iso]`.

## Uploading the ISO

Run `./bin/zyro-iso-upload [release/zyro.iso]`. This requires you've configured rclone (use `rclone config`).

## Full release of the ISO

Run `./bin/zyro-iso-release` to create, test, sign, and upload the ISO in one flow.
