# Publish the archive

1. Keep `conf/distributions`, `apt-repo.asc`, and `.github/workflows/publish.yml` in the repository. Set **Settings → Pages → Source** to **GitHub Actions**.
2. Add the armored private signing key as the Actions secret `APT_GPG_PRIVATE_KEY`. If it has a passphrase, add `APT_GPG_PASSPHRASE` as well. Set `SignWith` in each distribution block to the fingerprint of the matching public key.
3. Run **Actions → Publish APT repository → Run workflow**. The workflow fetches the four `HomeAutomationMonitorGW` packages from release `v1.1.5` and checks their accompanying `.deb.sha256` files. Select `both` to publish these packages for Debian 12 and 13.

Each deployment replaces the published archive. The workflow exports signed indexes for every distribution in `conf/distributions`. A distribution that receives no packages in that run has empty indexes. Before deployment, the workflow checks each distribution with `apt-get update`, downloads the indexed packages with APT, and compares them with the release files.

