# rw-web
TODO: cleanup publisher homepage generator.

```
$ ./bin/radiowitness json dat://author.key \
    --studio dat://studio.key \
    --index dat://db.key \
    > web/dat.json
$ npm run build --prefix web/
```

## Commands
Command                | Description                                      |
-----------------------|--------------------------------------------------|
`$ npm start`          | Start the development server
`$ npm test`           | Lint, validate deps & run tests
`$ npm run build`      | Compile all files into `dist/`
`$ npm run create`     | Generate a scaffold file
`$ npm run inspect`    | Inspect the bundle's dependencies
