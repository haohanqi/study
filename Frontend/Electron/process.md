# Main Process and Render Process in Electron

Coding in Electron is different from coding in React project. We need to understand two different part: **Render part** just our frontend code, it could be react or simple javascript and html. **Main process part** is like a "backend" for the frontend, help us to handle electron native event, create window...

The main difficulty is to understand:

1. What is `main process` and `render process`
2. How does two process communicate with each other

## What is `main process`

> Main process is like a node backend listen to several native event, or event dispatch from frontend.

**It's a node process**
It may be confused when a project has `nodejs` `require` and `es6` `import` at same time. But `main process` is a node process, even though electron support `es6` import, we do need to use node module such as `path`, `process.cwd()`.
