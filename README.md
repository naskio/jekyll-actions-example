# jekyll-actions-example

```shell
pip install -r Site/scripts/requirements.txt
cd Site/
bundle install
bundle exec jekyll clean
bundle exec jekyll serve --config _config.yml,_config.development.yml
bundle exec jekyll build --config _config.yml,_config.development.yml
```