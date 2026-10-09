---
name: publish
description: Use when publishing a new version of ccfaststatus — covers bump, tag, GitHub Actions release build, and Homebrew tap formula update on ronalove/homebrew-tap
---

# publish

Publier une nouvelle version de ccfaststatus : bump version, release GitHub
(tarball + sha256 via Actions), MAJ du Formula dans le tap Homebrew.

## Quand utiliser

- L'utilisateur demande « publier », « release vX.Y.Z », « nouvelle version », « publish »
- Tout le code à sortir est déjà commité sur `main`

## Paramètres du projet (figés)

| Champ | Valeur |
|-------|--------|
| Repo principal | `ronalove/ccfaststatus` |
| Repo tap | `ronalove/homebrew-tap` |
| Formula | `Formula/ccfaststatus.rb` |
| Cible build | `aarch64-apple-darwin` (macOS arm64 uniquement) |
| Nom binaire | `ccfaststatus` |
| Workflow CI | `.github/workflows/release.yml` (trigger : tag `v*`) |

## Vue d'ensemble

1. Bump `version` dans `Cargo.toml` + commit
2. Tag annoté `vX.Y.Z` + push (branche **et** tag)
3. GitHub Actions build macOS arm64 → Release avec tarball + `.sha256`
4. MAJ Formula dans `ronalove/homebrew-tap` (version, url, sha256)
5. Vérification `brew upgrade`

## Étape 1 — Bump version

Remplacer `version = "..."` (ligne unique) dans `Cargo.toml`. Puis :

```sh
cargo check --quiet   # met à jour Cargo.lock
git add Cargo.toml Cargo.lock
git commit -m "chore(release): bump to vX.Y.Z"
```

## Étape 2 — Tag + push

```sh
git tag -a vX.Y.Z -m "vX.Y.Z — <résumé>"
git push origin main
git push origin vX.Y.Z          # tag poussé séparément
```

**Gotcha :** `git push origin main` ne pousse **pas** les tags. Les deux push
sont obligatoires, sinon le workflow Release ne se déclenche pas.

## Étape 3 — Attendre la Release

```sh
RUN_ID=$(gh run list --repo ronalove/ccfaststatus --workflow Release --limit 1 \
         --json databaseId -q '.[0].databaseId')
gh run watch "$RUN_ID" --repo ronalove/ccfaststatus --exit-status
```

Durée typique : ~1 min cache cargo chaud, ~3 min cache froid.

## Étape 4 — Récupérer le SHA256

```sh
VERSION=X.Y.Z
SHA=$(curl -fsSL \
  "https://github.com/ronalove/ccfaststatus/releases/download/v${VERSION}/ccfaststatus-${VERSION}-aarch64-apple-darwin.tar.gz.sha256" \
  | awk '{print $1}')
echo "$SHA"
```

Le fichier `.sha256` contient `<hash>  <filename>` — on ne garde que le hash.

## Étape 5 — MAJ Formula dans le tap

```sh
cd /tmp && rm -rf homebrew-tap
git clone https://github.com/ronalove/homebrew-tap.git
cd homebrew-tap
```

Éditer `Formula/ccfaststatus.rb` — **3 remplacements** :
- `version "X.Y.Z"`
- URL : `/download/vX.Y.Z/ccfaststatus-X.Y.Z-aarch64-apple-darwin.tar.gz`
  (la version apparaît **deux fois** dans l'URL : `/vX.Y.Z/` puis `-X.Y.Z-`)
- `sha256 "<nouveau-hash>"`

`url` et `sha256` restent au niveau de la classe, jamais dans un bloc
`on_macos` : sous Linux, la formule n'aurait plus d'URL et `brew tap
ronalove/tap` refuserait tout le tap. `depends_on :macos` et
`depends_on arch: :arm64` refusent déjà l'installation ailleurs.

Avant de pousser, la vérification que fait `brew tap` (chaque formule doit se
charger sur tous les systèmes et processeurs connus de Homebrew) :

```sh
brew ruby -e '
  require "readall"
  files = ARGV.map { |file| Pathname(file).expand_path }
  exit Readall.valid_tap?(Tap.fetch("ronalove/tap"), formula_files: files, cask_files: [])
' Formula/*.rb && echo "tap valide"
```

Sans « tap valide », ne pas pousser : brew affiche la formule et la plateforme
en cause.

```sh
git add Formula/ccfaststatus.rb
git commit -m "ccfaststatus: vX.Y.Z"
git push origin main
```

## Étape 6 — Vérifier sur le poste

```sh
brew update                        # resync du tap
brew upgrade ccfaststatus
echo '{}' | ccfaststatus | head -c 40
```

## Gotchas à surveiller

| Piège | Solution |
|-------|----------|
| Tag non poussé après push main | Deux push distincts : `main` puis `vX.Y.Z` |
| `Cargo.lock` pas à jour dans le commit de bump | `cargo check --quiet` avant `git add` |
| Version oubliée dans l'URL du Formula | Remplacer aux **deux** endroits (`/vX.Y.Z/` + `-X.Y.Z-`) |
| Mauvais format SHA256 copié | `awk '{print $1}'` sur le fichier `.sha256` |
| `brew install` voit encore l'ancienne version | `brew update` force la resync du tap |
| Workflow CI `Release` ne démarre pas | Vérifier que le tag matche le pattern `v*` |
| `brew tap ronalove/tap` refuse tout le tap (« formula requires at least a URL ») | `url` et `sha256` hors de `on_macos`, vérification `brew ruby` de l'étape 5 avant le push (`brew install` et `brew update` ne la font pas) |
