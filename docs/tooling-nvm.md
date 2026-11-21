# Selecting a Node version with nvm

The repository records the Node version for local development in
[`.nvmrc`](../.nvmrc) (currently `24`). `package.json` also declares
`"engines": { "node": ">=24.15.0" }`, so use a 24.x release that meets that
minimum. Installing from `.nvmrc` selects the latest available 24.x release,
which satisfies that minimum.

Run these commands from the repository root so nvm can find `.nvmrc`:

```bash
# Switch to the version recorded in .nvmrc
nvm use

# If that version is not installed yet, install it first
nvm install
nvm use
```

`nvm use` reads `.nvmrc` when no version is passed. Likewise, `nvm install` with
no argument installs the version recorded in `.nvmrc`; `nvm install 24` is
equivalent here. If `nvm use` reports that the version is not yet installed,
run `nvm install` from the same directory and try again.

Verify the active version before installing dependencies:

```bash
node -v
npm install
```
