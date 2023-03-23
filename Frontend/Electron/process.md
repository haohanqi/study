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

## What is render process

#

> Render process will be the frontend, we can use different frontend framework such as React | Vue.

**How to use React + Electron**

Step 1: Decide use which framework as your based framework

It could be use electron as based framework, then try to figure out how to make React fit into it, or we can do other way around.

Recommend to use React as based framework, since we don't need to worry about webpack for React build(render process build), CRA will help us to do it. 

Step 2: Create react app

Step 3: Install electron-forge

```js
npm install --save-dev @electron-forge/cli

npx electron-forge import
```

Should see following changes in `package.json`
```json
"start": "electron-forge start",
"package": "electron-forge package",
"make": "electron-forge make"
```

Step 4: Make changes in `index.ts`

**Change web view load path**

Since in the development mode, we are loading frontend from `localhost:3000`, but in production mode we are loading it from build `index.html`
```js
mainWindow.loadURL(isDev
      ? "http://localhost:3000"
      : `file://${path.join(__dirname, "../../build/index.html")}`,
);
```

**Load compiled preload.js file**

```js
 mainWindow = new BrowserWindow({
    width: electronScreen.getPrimaryDisplay().workArea.width,
    height: electronScreen.getPrimaryDisplay().workArea.height,
    show: false,
    backgroundColor: "white",
    webPreferences: {
      preload: path.join(
        __dirname,
        "../../.electronMainProcessWebpack/preload/index.js",
      ),
    },
  });
  ```


Step 5: Start Script
```json
"dev-windows-start": "cross-env concurrently \"run-s build-electron-main dev\" \"wait-on http://localhost:3000 && npm run electron-app-start\"",
```

## Communication between Main Process and Render Process
#
Because of security reason, we can't directly import electron method into frontend. Need to use `preload` to expose all electron methods to frontend.

**Make sure in the `index.ts`, when we create main window, nodeIntegration is `false`**

Example of `preload`
```js
import { contextBridge, ipcRenderer, IpcRendererEvent } from "electron";

contextBridge.exposeInMainWorld("electron", {
  isWindows: process.platform === "win32",
  methodChannel: {
    test: () => {
      console.info("Preload test method called!");
    },
    invoke: (channel: string, ...args: unknown[]): Promise<any> =>
        ipcRenderer.invoke(channel, ...args),
    send: (channel: string, ...args: unknown[]): void =>
      ipcRenderer.send(channel, ...args),
    on: (
      channel: string,
      listener: (event: IpcRendererEvent, ...args: unknown[]) => void,
    ): void => {
      ipcRenderer.on(channel, listener);
    },
    once: (
      channel: string,
      listener: (event: IpcRendererEvent, ...args: unknown[]) => void,
    ): void => {
      ipcRenderer.once(channel, listener);
    },
  },
});
```

Now we can access electron method by `window.electron.methodChannel`