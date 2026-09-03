# s5dsn-eqee.github.io

Personal site: business card + dev notes. Built with [Hugo](https://gohugo.io)
and [PaperMod](https://github.com/adityatelange/hugo-PaperMod), deployed to
GitHub Pages via GitHub Actions on every push to `main`.

## Local preview

```bash
git clone --recurse-submodules git@github.com:s5dsn-eqee/s5dsn-eqee.github.io.git
hugo server -D   # http://localhost:1313
```

## New note

```bash
hugo new content/notes/my-note.md
```

Set `draft: false` in the front matter when it's ready, then push.
