# Publish the archive

The central repository is `tseiman/apt-repro`. Application projects publish Debian packages as GitHub Release assets. The central workflow collects those assets, builds and signs the APT archive, and deploys it to GitHub Pages.

[Browse the published archive](https://tseiman.github.io/apt-repro/)

## Set up the central repository

1. Keep `conf/distributions`, `conf/sources.json`, `apt-repo.asc`, and `.github/workflows/publish.yml` on the default branch. Set **Settings → Pages → Source** to **GitHub Actions**.
2. Add the armored private signing key as the repository Actions secret `APT_GPG_PRIVATE_KEY`. If it has a passphrase, add `APT_GPG_PASSPHRASE` as well. Commit the corresponding public key as `apt-repo.asc` and set `SignWith` in each distribution block to its full fingerprint without spaces.
3. Define the distribution codenames, architectures, and components in `conf/distributions`. The current archive uses `bookworm` (Debian 12), `trixie` (Debian 13), `amd64`, `arm64`, and the `main` component.

## Add an application project

Publish the `.deb` files in a public GitHub Release. Give each package a matching checksum asset named `<filename>.deb.sha256`. The checksum file may contain just the SHA256 digest or the output of `sha256sum`, with the digest first.

Add an entry to `conf/sources.json`. List each Debian distribution on which those builds work.

```json
[
  {
    "repository": "tseiman/HomeAutomationMonitorGW",
    "distributions": ["bookworm", "trixie"]
  }
]
```

By default, the workflow downloads every `.deb` asset from the repository's latest published GitHub Release. Use the optional `tag` field to pin a release, such as `"tag": "v1.1.5"`. Use the optional `asset_pattern` field to select package filenames by glob. If a release contains separate Debian 12 and Debian 13 builds, add one entry per build set with its own pattern and distribution. [GitHub CLI release download reference](https://cli.github.com/manual/gh_release_download)

## Create the dispatch token

1. On GitHub, open your account **Settings → Developer settings → Personal access tokens → Fine-grained tokens → Generate new token**. [Open the token creation form](https://github.com/settings/personal-access-tokens/new)
2. Enter a name such as `apt-repro-dispatch` and choose an expiration date. Set **Resource owner** to `tseiman`.
3. Under **Repository access**, choose **Only select repositories** and select `apt-repro`. Under **Repository permissions → Contents**, select **Read and write**. This is the permission required by the dispatch endpoint. [GitHub dispatch API permissions](https://docs.github.com/en/rest/repos/repos#create-a-repository-dispatch-event)
4. Click **Generate token** and copy its value. In the **application repository**, open **Settings → Secrets and variables → Actions → New repository secret**. Set the name to `APT_REPO_DISPATCH_TOKEN`, paste the token into **Secret**, and click **Add secret**. Replace the secret with a fresh token when it expires. [GitHub Actions secrets](https://docs.github.com/en/actions/how-tos/write-workflows/choose-what-workflows-do/use-secrets)

## Trigger publishing from an application workflow

Add this step to the application's release job after the release is published and all package and checksum files have been uploaded. If separate build jobs upload files for different architectures, run this step after all upload jobs succeed.

```yaml
- name: Trigger central APT repository
  env:
    GH_TOKEN: ${{ secrets.APT_REPO_DISPATCH_TOKEN }}
  run: gh api --method POST repos/tseiman/apt-repro/dispatches -f event_type=apt-release-published
```

The dispatch requests a rebuild. Track the deployment in **Actions → Publish APT repository** in `tseiman/apt-repro`. You can also start a rebuild there with **Run workflow**.

Each deployment replaces the published archive with a fresh build from all entries in `conf/sources.json`. Every distribution in `conf/distributions` gets signed indexes, including empty indexes for distributions without packages. Before deployment, the workflow runs `apt-get update` against each distribution, downloads the indexed packages with APT, and compares them with the release files.

## Register the archive and test installation

On Debian, download the public key and verify its fingerprint.

```sh
curl --fail --location --output apt-repo.asc https://tseiman.github.io/apt-repro/apt-repo.asc
gpg --show-keys --fingerprint apt-repo.asc
```

The expected fingerprint is `FBB4 E68D ABB4 A25F 6E73 C81D D4EA 266E 8BC8 043F`. If it matches, install the key and register the archive. Set `suite=trixie` for Debian 13.

```sh
suite=bookworm
sudo install -d -m 0755 /etc/apt/keyrings
sudo install -m 0644 apt-repo.asc /etc/apt/keyrings/tseiman-apt-repro.asc
printf 'Types: deb\nURIs: https://tseiman.github.io/apt-repro/\nSuites: %s\nComponents: main\nSigned-By: /etc/apt/keyrings/tseiman-apt-repro.asc\n' "$suite" | sudo tee /etc/apt/sources.list.d/tseiman-apt-repro.sources > /dev/null
```

Replace `PACKAGE_NAME` with a published package. Check that `apt-cache policy` shows the central archive, then simulate and install the package.

```sh
sudo apt-get update
apt-cache policy PACKAGE_NAME
sudo apt-get --simulate install PACKAGE_NAME
sudo apt-get install PACKAGE_NAME
```

For a migration from a project-specific archive, disable its old APT source after verifying the central archive. Ubuntu compatibility has not been verified.
