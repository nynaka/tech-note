# Website

This website is built using [Docusaurus](https://docusaurus.io/), a modern static website generator.

## 起動方法

**git clone** 後、下記の手順で起動します。  
① は clone 後に一度実行し、それ以降は実行不要です。

1. Docusaurus のインストール

    ```bash
    npm install @docusaurus/core@latest @docusaurus/preset-classic@latest
    ```

2. Docusaurus の起動

    ```bash
    npm start
    ```


## Installation

```bash
yarn
```

## Local Development

```bash
yarn start
```

This command starts a local development server and opens up a browser window. Most changes are reflected live without having to restart the server.

## Build

```bash
yarn build
```

This command generates static content into the `build` directory and can be served using any static contents hosting service.

## Deployment

Using SSH:

```bash
USE_SSH=true yarn deploy
```

Not using SSH:

```bash
GIT_USER=<Your GitHub username> yarn deploy
```

If you are using GitHub pages for hosting, this command is a convenient way to build the website and push to the `gh-pages` branch.
