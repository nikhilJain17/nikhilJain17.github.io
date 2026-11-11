# Nikhil Jain's blog

Personal website built with Hugo and the PaperMod theme.

## Preview changes locally

Run these commands from the repository root:

```sh
hugo server -D --disableFastRender
```

Open http://localhost:1313/ (or the URL printed by Hugo). Keep the terminal
running while editing; saving files rebuilds the site and reloads the browser.
Press `Ctrl+C` to stop the server.

- `-D` includes posts with `draft: true` in their front matter.
- `--disableFastRender` fully re-renders pages when files change.
- Add `-F` to preview posts with future publication dates.
- If port 1313 is occupied, use `hugo server -D --disableFastRender --port 1314`.

Hugo previews saved files in the working directory, including untracked files;
you do not need to stage, commit, or push changes to see them locally.
Posts live in `content/notes/`. For example,
`content/notes/wllama_streaming_memory_reduction.md` is available at
http://localhost:1313/notes/wllama_streaming_memory_reduction/ unless its front
matter overrides the URL.

### If Hugo hangs before the preview starts

On macOS, cloud storage can offload files in a synced Documents folder. These
files still appear in Finder but have no local contents (`ls -lO` shows
`dataless`), and Hugo can stall while macOS tries to download them. In Finder,
choose **Keep Downloaded** for this repository if that option is available,
or keep the checkout outside cloud-synced folders (for example, `~/Developer`).
Wait for downloads to finish, then restart Hugo.

## Setup on a new machine

Install Hugo (extended edition). On macOS with Homebrew:

```sh
brew install hugo
```

Initialize the PaperMod theme submodule after cloning:

```sh
git submodule update --init --recursive
```

Site settings are in `hugo.toml`; custom layouts and styles are in `layouts/`
and `assets/`.

## Production build and publishing

Build locally with the same environment and minification used by CI:

```sh
hugo --environment production --minify
```

Generated output goes to `public/`, which is ignored by Git. Draft posts are
excluded from this build; use `draft: true` in front matter for unfinished posts.

Pushing to `master` triggers `.github/workflows/hugo.yaml` to build and deploy
the site to GitHub Pages. That workflow currently pins Hugo extended 0.146.0;
if local and deployed output differ, check `hugo version` against the CI version.
