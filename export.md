# Export Markdown


## Export to PDF
```shell
docker run --rm --init -v $PWD:/home/marp/app/ -e LANG=$LANG marpteam/marp-cli README.md --pdf --allow-local-files
```

## Export to PPTX
```shell
docker run --rm --init -v $PWD:/home/marp/app/ -e LANG=$LANG marpteam/marp-cli README.md --pptx --allow-local-files
```