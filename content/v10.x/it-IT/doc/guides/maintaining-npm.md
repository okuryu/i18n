# Manutenzione di npm in Node.js

New pull requests should be opened when a "next" version of npm has been released. Once the "next" version has been promoted to "latest" the PR should be updated as necessary.

Two weeks after the "latest" release has been promoted it can land on master assuming no major regressions are found. There are no additional constraints for Semver-Major releases.

The specific Node.js release streams the new version will be able to land into are at the discretion of the release and LTS teams.

This process only covers full updates to new versions of npm. Cherry-picked changes can be reviewed and landed via the normal consensus seeking process.

## Step 1: Clonare npm

```console
$ git clone https://github.com/npm/cli.git npm
$ cd npm
```

o se hai già un npm clonato assicurati che il repository sia aggiornato

```console
$ git remote update -p
$ git reset --hard origin latest
```

## Step 2: Costruire la release

```console
$ git checkout vX.Y.Z
$ make release
```

Note: please run `npm dist-tag ls npm` and make sure this is the `latest` **dist-tag**. `latest` on git is usually released as `next` when it's time to downstream

## Step 3: Rimuovere il vecchio npm

```console
$ cd /path/to/node
$ git remote update -p
$ git checkout -b npm-x.y.z origin/master
$ cd deps
$ rm -rf npm
```

## Step 4: Estrarre ed eseguire il commit sul nuovo npm

```console
$ tar zxf /path/to/npm/release/npm-x.y.z.tgz
$ git add -A npm
$ git commit -m "deps: upgrade npm to x.y.z"
$ cd ..
```

## Step 5: Aggiornare le licenze

```console
$ ./configure
$ make -j4
$ ./tools/license-builder.sh
# I comandi seguenti sono necessari esclusivamente se sono presenti modifiche
$ git add .
$ git commit -m "doc: update npm LICENSE using license-builder.sh"
```

Nota: assicurati di effettuare esclusivamente gli aggiornamenti che vengono modificati da npm.

## Step 6: Applicare Whitespace fix

```console
$ git rebase --whitespace=fix master
```

## Step 7: Testare la build

```console
$ make test-npm
```
