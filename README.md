# How to use the generator

### Testing
In a terminal run `hugo server -D`.
 - Open the `localhost:1313` link it prints to the terminal.
 - Any changes to .md content files are updated in real time.

### New page

Run this from the root directory to start a new post:
`hugo new content <path/to/new_post_file.md>`

### Publishing to Github pages
In a terminal, run `hugo serve`. This produces the static site pages to the `public/` directory.

Double check that the `public/` directory is symlinked to the https://github.com/30Wedge/30Wedge.github.io repo.

`cd public`, then commit all and push.

# Interaction with content

Hi, this is where I'm hosting the source for my articles.

If you see a mistake in one of these articles, or want to add a comment, please send me a pull request to this repo.
