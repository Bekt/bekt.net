## bekt.net [![](https://api.netlify.com/api/v1/badges/72b359d9-9e21-4ed5-a386-e9c750e83d44/deploy-status)](https://app.netlify.com/sites/bekt-net-fa9d/deploys)

Personal website.

## Development

```sh
$ docker run --rm --volume="$PWD:/srv/jekyll" -p 4000:4000 -it jekyll/jekyll:4.2.0 jekyll serve --livereload
```

## Writing

a. Manually add an .md file to _posts

b. Manage content via bekt.net/admin. This is managed by [Netlify CMS](https://www.netlifycms.org/docs/intro/) which auto-commits changes back to the repo.

## Debugging

- Namecheap - domain registrar
- Cloudflare - DNS
- Netlify - CDN
- Github - code + deploy trigger
- Jekyll - site generation
- Netlify CMS - UI for content publishing
