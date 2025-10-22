# zenn-contents

https://zenn.dev/kot149

## Setup

```sh
bun install
```

Install [textlint VSCode extension](https://marketplace.visualstudio.com/items?itemName=3w36zj6.textlint)

## Creating a new article

```sh
bun new
```

Or with slug specified:

```sh
bun new --slug my-awesome-article
```

## Preview an article

```sh
bun dev
```

## Zenn-specific Markdown Syntax

### Message block (Yellow background with info icon)

```markdown
:::message
This is a message.
:::
```

### Important Message Block (Red background with warning icon)

```markdown
:::message alert
This is an important message.
:::
```

### Toggleable content block

```markdown
:::details Title
This is a details.
:::
```

### Nested message blocks

Add more colons to parent block to nest message blocks.

```markdown
::::message
This is a message.
:::
:::message alert
This is an important message.
::::
```
