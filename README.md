# Standalone nx workspace for a publishable typescript library

This repo contains an example nx workspace containing a standalone typescript library which can be published using the new `nx release` command.

Below are my steps taken to configure this project.

## Create workspace

```sh
npx create-nx-workspace@latest \
    --preset ts-standalone \
    --ci skip \
    my-lib
```

## Manual edits

- Change package name in `package.json` to `@my-org/my-lib`
- Also updated name of package in other files.

## Publish

Below steps describe how I edited the workspace to properly use
the `nx release publish` command to publish this library to the npm registry.

### Publish

```sh
❯ npx nx release publish --dry-run --verbose

 >  NX   Running target nx-release-publish for project my-lib:

    - my-lib

   With additional flags:
     --dryRun=true

 ———————————————————————————————————————————————————————————————————————————————————————————————————————————————

> nx run my-lib:nx-release-publish


📦  @my-org/my-lib@0.0.1
=== Tarball Contents ===

13B  .eslintignore
577B .eslintrc.json
83B  .prettierignore
26B  .prettierrc
76B  .vscode/extensions.json
308B README.md
763B nx.json
976B package.json
732B project.json
48B  src/index.ts
222B src/lib/my-lib.spec.ts
89B  src/lib/my-lib.ts
622B tsconfig.json
245B tsconfig.lib.json
499B tsconfig.spec.json
655B vite.config.ts
=== Tarball Details ===
name:          @my-org/my-lib
version:       0.0.1
filename:      my-org-my-lib-0.0.1.tgz
package size:  2.5 kB
unpacked size: 5.9 kB
shasum:        46dad410ec2f00c51eab1f7020d527cd2a921890
integrity:     sha512-TVTR8LzJM4MPs[...]5Kwgc5l8FWv7g==
total files:   16

Would publish to https://registry.npmjs.org/ with tag "latest", but [dry-run] was set

 ———————————————————————————————————————————————————————————————————————————————————————————————————————————————

 >  NX   Successfully ran target nx-release-publish for project my-lib



NOTE: The "dryRun" flag means no changes were made.

```

We notice that all files from the workspace root are about to be published.
This is not desirable but can be resolved without blaming `nx` or the new `nx release` command.

So we now know that we need to limit the files being published.
To do this, we can set the `files` property in our `package.json` to only list the files and folders that should be published. In this case it could contain just the dist folder.

```json
"files": {
    "dist"
}
```

```sh
❯ npx nx release publish --dry-run --verbose

 >  NX   Running target nx-release-publish for project my-lib:

    - my-lib

   With additional flags:
     --dryRun=true

 ———————————————————————————————————————————————————————————————————————————————————————————————————————————————

> nx run my-lib:nx-release-publish


📦  @my-org/my-lib@0.0.1
=== Tarball Contents ===

2.2kB README.md
1.0kB package.json
=== Tarball Details ===
name:          @my-org/my-lib
version:       0.0.1
filename:      my-org-my-lib-0.0.1.tgz
package size:  1.3 kB
unpacked size: 3.2 kB
shasum:        0ddc259e1f38657888958a90dee2b85d70ec147a
integrity:     sha512-qb3YUdKrEiNnG[...]uw7xpsCf7byAQ==
total files:   2

Would publish to https://registry.npmjs.org/ with tag "latest", but [dry-run] was set

 ———————————————————————————————————————————————————————————————————————————————————————————————————————————————

 >  NX   Successfully ran target nx-release-publish for project my-lib
```

We now see that only the `package.json` and `README.md` file would be published.
Did we forget to build the package? Yes!

```sh
npx nx build
```

Let see if `nx release publish` now would publish only the required files.

```sh
❯ npx nx release publish --dry-run --verbose

 >  NX   Running target nx-release-publish for project my-lib:

    - my-lib

   With additional flags:
     --dryRun=true

 ——————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————

> nx run my-lib:nx-release-publish


📦  @my-org/my-lib@0.0.1
=== Tarball Contents ===

4.1kB README.md
1.0kB dist/package.json
4.1kB dist/README.md
48B   dist/src/index.d.ts
218B  dist/src/index.js
119B  dist/src/index.js.map
57B   dist/src/lib/my-lib.d.ts
300B  dist/src/lib/my-lib.js
215B  dist/src/lib/my-lib.js.map
1.0kB package.json
=== Tarball Details ===
name:          @my-org/my-lib
version:       0.0.1
filename:      my-org-my-lib-0.0.1.tgz
package size:  2.2 kB
unpacked size: 11.2 kB
shasum:        b5e0d91f7bc2a35e261af2981c074913ffb6a58c
integrity:     sha512-+h3LijX+IdPIC[...]nSAu/ZUUC4zzA==
total files:   10

Would publish to https://registry.npmjs.org/ with tag "latest", but [dry-run] was set
```

Hmm... two `package.json` files?

Yes, `nx` will generate a minimal `package.json` file together with your compiled javascript files.

Ideally we would run `npm publish` from within the `dist` folder but I didn't see
the official nx docs to change the `cwd` used by the `nx release publish` command.

Therefor we can ignore that generated `package.json` for now and use the `package.json` file from the our repo.

The only thing we have to adjust is the `main` property in `package.json` so that it contains the correct path of our compiled files.

```json
  "main": "./dist/src/index.js",
  "typings": "./dist/src/index.d.ts",
  "files": [
    "dist"
  ]
```

To avoid the `src` part we could also add `rootDir` property to the `compilerOptions` in our `tsconfig.lib.json`.

```
  "compilerOptions": {
    "outDir": "./dist/out-tsc",
    "declaration": true,
    "types": ["node"],
    "rootDir": "./src"
  },
```

**The above won't work ** because we are using a tsc executor from `nx` which will override this `rootDir` property anyway.
Luckily we can still pass the same option the the build target in our `project.json`.

```json
    "build": {
      "executor": "@nx/js:tsc",
      "outputs": ["{options.outputPath}"],
      "options": {
        "outputPath": "dist",
        "main": "./src/index.ts",
        "tsConfig": "./tsconfig.lib.json",
        "assets": ["*.md"],
        "rootDir": "./src"
      }
    },
```

Finally we can cleanup our `package.json` by removing the `src` sub directory from `main` and `typings`.

```json
  "main": "./dist/index.js",
  "typings": "./dist/index.d.ts",
  "files": [
    "dist"
  ]
```

```sh
❯ npx nx release publish --dry-run --verbose

 >  NX   Running target nx-release-publish for project my-lib:

    - my-lib

   With additional flags:
     --dryRun=true

 ——————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————

> nx run my-lib:nx-release-publish


📦  @my-org/my-lib@0.0.1
=== Tarball Contents ===

7.7kB README.md
48B   dist/index.d.ts
218B  dist/index.js
116B  dist/index.js.map
57B   dist/lib/my-lib.d.ts
300B  dist/lib/my-lib.js
212B  dist/lib/my-lib.js.map
1.0kB dist/package.json
7.2kB dist/README.md
1.0kB package.json
=== Tarball Details ===
name:          @my-org/my-lib
version:       0.0.1
filename:      my-org-my-lib-0.0.1.tgz
package size:  3.2 kB
unpacked size: 18.0 kB
shasum:        32590f9dabd30a79a82996fdd3c7be31149c76b0
integrity:     sha512-7u6Ew4cesr1tP[...]o4xbODH+cPjdg==
total files:   10

Would publish to https://registry.npmjs.org/ with tag "latest", but [dry-run] was set
```

We still see two `package.json` files, so let's exclude the generated `package.json` file using the `files`
property in `package.json`.

```json
  "files": [
    "dist",
    "!dist/package.json"
  ]
```

And tadaa!

```sh
❯ npx nx release publish --dry-run

 >  NX   Running target nx-release-publish for project my-lib:

    - my-lib

   With additional flags:
     --dryRun=true

 ———————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————

> nx run my-lib:nx-release-publish


📦  @my-org/my-lib@0.0.1
=== Tarball Contents ===

1.1kB  LICENSE
10.3kB README.md
30B    dist/index.d.ts
200B   dist/index.js
116B   dist/index.js.map
41B    dist/lib/my-lib.d.ts
200B   dist/lib/my-lib.js
172B   dist/lib/my-lib.js.map
10.6kB dist/README.md
1.0kB  package.json
=== Tarball Details ===
name:          @my-org/my-lib
version:       0.0.1
filename:      my-org-my-lib-0.0.1.tgz
package size:  4.3 kB
unpacked size: 23.8 kB
shasum:        759b65eeb426b540ea9d93f946c0854f352dcd4b
integrity:     sha512-KLpammVULAbED[...]xZDugQ6nIdk4w==
total files:   10

Would publish to https://registry.npmjs.org/ with tag "latest", but [dry-run] was set

 ———————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————

 >  NX   Successfully ran target nx-release-publish for project my-lib
```

### Version and Changelog

Additional configuration can be added to configure automatically perform a version bump based on conventional commit messages and to generate changelog updates on every release.

Add the `release` block to `nx.json`.

```json
"release": {
    "version": {
      "git": {
        "commit": false,
        "tag": false
      },
      "generatorOptions": {
        "specifierSource": "conventional-commits",
        "currentVersionResolver": "git-tag",
        "fallbackCurrentVersionResolver": "disk"
      }
    },
    "changelog": {
      "workspaceChangelog": true,
      "automaticFromRef": true,
      "git": {
        "commit": true,
        "tag": true
      }
    }
  },
```
