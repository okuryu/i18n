# Maintenir npm en Node.js

## Étape 1 : Cloner npm

```console
$ git clone https://github.com/npm/npm.git
$ cd npm
```

ou si vous avez déjà npm cloné, assurez-vous que le repo est à jour

```console
$ git remote update -p
$ git reset --hard origin latest
```

## Step 2: Build release

```console
$ git checkout vX.Y.Z
$ make release
```

Note: please run `npm dist-tag ls npm` and make sure this is the `latest` **dist-tag**. `latest` on git is usually released as `next` when it's time to downstream

## Step 3: Remove old npm

```console
$ cd /path/to/node
$ git remote update -p
$ git checkout -b npm-x.y.z origin/master
$ cd deps
$ rm -rf npm
```

## Step 4: Extract and commit new npm

```console
$ tar zxf /path/to/npm/release/npm-x.y.z.tgz
$ git add -A npm
$ git commit -m "deps: upgrade npm to x.y.z"
$ cd ..
```

## Étape 5: Mise à jour des licences

```console
$ ./configure
$ make -j4
$ ./tools/license-builder.sh
# The following commands are only necessary if there are changes
$ git add .
$ git commit -m "doc: update npm LICENSE using license-builder.sh"
```

Note: veuillez vous assurer que vous ne faites que les mises à jour qui sont modifiées par npm.

## Step 6: Apply Whitespace fix

```console
$ git rebase --whitespace=fix master
```

## Étape 7: Tester le build

```console
$ make test-npm
```
