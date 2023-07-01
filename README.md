# Jerome Ducret page

This is the codebase for my personnal webpage.
This repository is maintained with [Renovate](https://github.com/renovatebot/renovate)

## Local tests

```sh
# Install bundle
## in `docs` folder
docker run --rm -it -v `pwd`/:/workdir --network host --workidr workdir ruby bash
bundle install
bundle exec jekyll serve
```
