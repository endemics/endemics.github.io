# blog.endemics.info - hugo based blog

## Repository structure

```
endemics-blog/
├── .devcontainer/
│   ├── devcontainer.json
│   └── Dockerfile
├── .vscode/
│   └── tasks.json
├── .github/
│   └── workflows/
│       └── hugo.yml
├── static/
│   └── images/
├── content/
├── themes/
├── hugo.toml
└── .gitignore
```

## Devcontainer

Open in VS Code: Make sure you have the "Dev Containers" extension installed
Reopen in Container: Cmd+Shift+P → "Dev Containers: Reopen in Container"
The container will build and start automatically

Once inside the container, use the integrated terminal:
```bash
# Create new posts
hugo new content/posts/my-post.md

# Start the server (if not using postStartCommand)
hugo server --buildDrafts --buildFuture --bind 0.0.0.0

# Build for production
hugo --minify
```

The server will be accessible at http://localhost:1313

## Hugo commands

In vscode thanks to the task integration, use `Cmd+Shift+P` → "Tasks: Run Task" to easily run Hugo commands.
