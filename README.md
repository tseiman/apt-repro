# apt-repro

Central Tseiman Debian package archive hosted on GitHub Pages. [Browse the archive](https://tseiman.github.io/apt-repro/).

## Use the archive

The archive targets Debian 12 (`bookworm`) and Debian 13 (`trixie`) on `amd64` and `arm64`. Ubuntu compatibility has not been verified.

1. Download the public signing key and check its fingerprint.

   ```sh
   curl --fail --location --output apt-repo.asc https://tseiman.github.io/apt-repro/apt-repo.asc
   gpg --show-keys --fingerprint apt-repo.asc
   ```

   The expected fingerprint is `FBB4 E68D ABB4 A25F 6E73 C81D D4EA 266E 8BC8 043F`.

2. If the fingerprint matches, install the key and register the archive. Set `suite=trixie` for Debian 13.

   ```sh
   suite=bookworm
   sudo install -d -m 0755 /etc/apt/keyrings
   sudo install -m 0644 apt-repo.asc /etc/apt/keyrings/tseiman-apt-repro.asc
   printf 'Types: deb\nURIs: https://tseiman.github.io/apt-repro/\nSuites: %s\nComponents: main\nSigned-By: /etc/apt/keyrings/tseiman-apt-repro.asc\n' "$suite" | sudo tee /etc/apt/sources.list.d/tseiman-apt-repro.sources > /dev/null
   ```

3. Refresh the indexes and check that the package is listed from this archive before installing it.

   ```sh
   sudo apt-get update
   apt-cache policy automation-gateway automation-gateway-led
   sudo apt-get install automation-gateway
   ```

   If you previously used a project-specific archive, disable its old APT source once the new one works.

## Publish the archive

1. Keep `conf/distributions`, `apt-repo.asc`, and `.github/workflows/publish.yml` in the repository. Set **Settings → Pages → Source** to **GitHub Actions**.
2. Add the armored private signing key as the Actions secret `APT_GPG_PRIVATE_KEY`. If it has a passphrase, add `APT_GPG_PASSPHRASE` as well. Set `SignWith` in each distribution block to the fingerprint of the matching public key.
3. Run **Actions → Publish APT repository → Run workflow**. The workflow fetches the four `HomeAutomationMonitorGW` packages from release `v1.1.5` and checks their accompanying `.deb.sha256` files. Select `both` to publish these packages for Debian 12 and 13.

Each deployment replaces the published archive. The workflow exports signed indexes for every distribution in `conf/distributions`. A distribution that receives no packages in that run has empty indexes. Before deployment, the workflow checks each distribution with `apt-get update`, downloads the indexed packages with APT, and compares them with the release files.

