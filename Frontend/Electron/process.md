# Main Process and Render Process in Electron

Coding in Electron is different from coding in React project. We need to understand two different part: **Render part** just our frontend code, it could be react or simple javascript and html. **Main process part** is like a "backend" for the frontend, help us to handle electron native event, create window...

The main difficulty is to understand:

1. What is `main process` and `render process`
2. How does two process communicate with each other

## What is main process

#

> Main process is like a node backend listen to several native event, or event dispatch from frontend.

**It's a node process**
It may be confused when a project has `nodejs` `require` and `es6` `import` at same time. But `main process` is a node process, even though electron support `es6` import, we do need to use node module such as `path`, `process.cwd()`.

If using typescript for the main process, one thing to notice is **main process entry file has to be javascript**, which means we need to compile and bundle main process, then set `main` property in `package.json` to javascript file.

For example we need a webpack  `electron_main_config.js`
```js
[
// this is for index.ts
  {
    entry: './src/electron/index.ts',
    target: 'electron-main',
    node: {
      __dirname: false,
      __filename: false,
    },
    devtool: 'inline-source-map',
    module: {
      rules: [
        {
          test: /\.tsx?$/,
          exclude: /(node_modules)/,
          use: [
            {
              loader: "babel-loader",
              options: {
                cacheDirectory: true,
                presets: [
                  "@babel/preset-env",
                  ["@babel/preset-typescript", { isTSX: true, allExtensions: true }],
                ],
              },
            },
          ],
        },
      ],
    },
    resolve: {
      extensions: [".js", ".ts", ".jsx", ".tsx"],
    },
    plugins: [
      new webpack.DefinePlugin({
        SENTRY_DSN: JSON.stringify(sentryDsn),
        APP_VERSION: JSON.stringify(appVersion),
      }),
    ],
    output: {
      filename: 'index.js',
      path: path.resolve(__dirname, '..', '..', 'build', 'electron', 'electron'),
    },
  },
  // This for preload.ts
  {
    entry: './src/electron/preload.ts',
    target: 'electron-preload',
    devtool: 'inline-source-map',
    module: {
      rules: [
        {
          test: /\.tsx?$/,
          exclude: /(node_modules|webpack)/,
          use: [
            {
              loader: "babel-loader",
              options: {
                cacheDirectory: true,
                presets: [
                  "@babel/preset-env",
                  ["@babel/preset-typescript", { isTSX: true, allExtensions: true }],
                ],
              },
            },
          ],
        },
      ],
    },
    resolve: {
      extensions: [".js", ".ts", ".jsx", ".tsx"],
    },
    output: {
      filename: 'preload.js',
      path: path.resolve(__dirname, '..', '..', 'build', 'electron', 'electron'),
    },
  },
];
```