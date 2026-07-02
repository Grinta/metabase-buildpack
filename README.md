Heroku Buildpack for Metabase

Add the following to your app.json:

"buildpacks": [
  {
    "url": "https://github.com/grinta/metabase-buildpack"
  }
]

The Metabase version is pinned in `bin/version`. To upgrade it, follow [MIGRATION.md](MIGRATION.md).
