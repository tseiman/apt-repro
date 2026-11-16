# apt-repro

Central Tseiman Debian package archive.

[Browse packages and archive files](https://tseiman.github.io/apt-repro/)

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

3. Refresh the indexes and install a package. Replace `PACKAGE_NAME` with the name of the package you want.

   ```sh
   sudo apt-get update
   apt-cache policy PACKAGE_NAME
   sudo apt-get --simulate install PACKAGE_NAME
   sudo apt-get install PACKAGE_NAME
   ```

   If you previously used a project-specific archive, disable its old APT source once the new one works.

