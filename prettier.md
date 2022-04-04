# 💅 Prettier

## Overview
- [Configuring Prettier VSCode Extension for directories](#configuring-prettier-vscode-extension-for-directories)

## Configuring Prettier VSCode Extension for directories
Prettier can be helpful for code formatting, to even allow VSCode to format your code on save.

A `.prettierrc` file can be added to your project to detail what is expected of the style of the code to keep to.

```json
{
  "trailingComma": "es5",
  "tabWidth": 2,
  "semi": true,
  "singleQuote": false,
  "jsxSingleQuote": false
}
```

Using the [Prettier - Code Formatter](https://marketplace.visualstudio.com/items?itemName=esbenp.prettier-vscode) extension, this can be used to format your code on save. By default, it will format the code to the style specified in the `./.prettierrc` file and ignore any files to style in `./.prettierignore`.

For use in directories, a `.vscode/settings.json` can be added at the top level with the following key-value pairs. The following will activate the format-on-save feature, as well as specify the `.prettierrc` and `.prettierignore` file to use should we wish to place them elsewhere.

```json
{
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "editor.formatOnSave": true,
  "prettier.configPath": "path/to/.prettierrc",
  "prettier.ignorePath": "path/to/.prettierignore"
}
```