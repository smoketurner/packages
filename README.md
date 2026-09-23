# Smoke Turner Packages

APT and YUM/DNF repositories for [Smoke Turner](https://github.com/smoketurner) projects, served at [packages.smoketurner.com](https://packages.smoketurner.com).

| Package | Types | Architectures |
|---------|-------|---------------|
| [`quack`](https://github.com/smoketurner/quack) | `.deb`, `.rpm` | amd64/x86_64, arm64/aarch64 |

## Installation

### APT (Debian / Ubuntu)

```bash
curl -fsSL https://packages.smoketurner.com/gpg/smoketurner.asc \
  | gpg --dearmor \
  | sudo tee /usr/share/keyrings/smoketurner-archive-keyring.gpg > /dev/null

echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/smoketurner-archive-keyring.gpg] https://packages.smoketurner.com/apt stable main" \
  | sudo tee /etc/apt/sources.list.d/smoketurner.list > /dev/null

sudo apt-get update && sudo apt-get install -y quack
```

### YUM / DNF (Fedora / RHEL)

```bash
sudo tee /etc/yum.repos.d/smoketurner.repo << 'EOF'
[smoketurner]
name=Smoke Turner
baseurl=https://packages.smoketurner.com/rpm/$basearch/
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.smoketurner.com/gpg/smoketurner.asc
enabled=1
EOF

sudo dnf install -y quack
```

## GPG key

RPM packages and all repository metadata are signed with [`gpg/smoketurner.asc`](gpg/smoketurner.asc), fingerprint `F004 4C01 F2D0 4F5E 44E5 18FE 84C0 D1A3 4448 CC13` (expires 2029-09-22). The private key and passphrase are the `GPG_PRIVATE_KEY` and `GPG_PASSPHRASE` Actions secrets on each publishing project, set by smoketurner-infra's `environments/github` root.

## How releases land here

Nothing in this repository builds packages. A project's release workflow builds and signs its `.deb` and `.rpm` files, then checks out this repository with a fine-grained token (`PACKAGES_REPO_TOKEN`, contents: write on this repository only) and:

1. copies `.deb` files into `apt/pool/main/` and `.rpm` files into `rpm/x86_64/` or `rpm/aarch64/`
2. regenerates `apt/dists/stable/` with `dpkg-scanpackages` and `apt-ftparchive release -c apt-ftparchive.conf`, then signs `Release` into `Release.gpg` and `InRelease`
3. runs `createrepo_c --update` for each RPM architecture and signs `repodata/repomd.xml`
4. commits `<project> <version>` and pushes to `main`

Because that push uses a personal access token rather than `GITHUB_TOKEN`, it triggers [`publish-to-s3.yml`](.github/workflows/publish-to-s3.yml), which syncs the repository to S3 and invalidates CloudFront.

Metadata is regenerated over the whole pool, so every project's packages stay indexed whichever project releases.

## Layout

```
apt/pool/main/          .deb packages
apt/dists/stable/       APT metadata (Release, InRelease, Packages)
rpm/x86_64/             x86_64 .rpm packages and repodata/
rpm/aarch64/            aarch64 .rpm packages and repodata/
gpg/smoketurner.asc     public signing key
apt-ftparchive.conf     Release file fields
index.html              landing page
```
