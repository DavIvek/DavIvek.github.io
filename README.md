# DavIvek.github.io

Personal engineering blog at <https://davivek.github.io>.

Built on the [Chirpy](https://github.com/cotes2020/jekyll-theme-chirpy) Jekyll theme.

## Adding a new post

1. Create a file under `_posts/` named `YYYY-MM-DD-slug.md` (the date prefix is required by Jekyll).
2. Start it with Chirpy front matter:

   ```yaml
   ---
   title: Your post title here
   date: 2026-06-01 10:00:00 +0200
   categories: [Engineering]
   tags: [some, tags, here]
   description: One-sentence summary used for SEO and link previews.
   ---
   ```

3. Write the post in markdown below the front matter. Don't include a top-level `# heading` — Chirpy renders the title from the `title:` field.
4. Commit and push to `main`. GitHub Pages rebuilds and redeploys within ~1 minute.

## Previewing locally (optional)

GitHub Pages will happily build the site for you, but if you want a preview before pushing:

```bash
# One-time: install Ruby + bundler (Ubuntu/Debian)
sudo apt install ruby-full build-essential

# In this directory:
bundle install
bundle exec jekyll serve
# → site available at http://127.0.0.1:4000
```

## Publishing for the first time

1. Create the remote repo: `gh repo create DavIvek.github.io --public --source . --remote origin`
2. Push: `git push -u origin main`
3. In the repo's **Settings → Pages**, set source to "GitHub Actions" (Chirpy ships a CD workflow that handles the build).
4. Wait ~1 minute, then visit <https://davivek.github.io>.

## TODOs before first publish

- Pick a tagline and description in `_config.yml` (currently placeholder).
- Fill in your LinkedIn URL in `_config.yml` under `social.links`.
- Optionally set `avatar:` to a URL/path of your profile picture.
- Optionally set `social_preview_image:` so LinkedIn link previews show a real image.
