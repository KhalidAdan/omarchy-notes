# Node.js Quick Setup Runbook for Omarchy

## Installation

### Standard Install (Recommended)
```bash
# Install Node.js and npm
sudo pacman -S nodejs npm

# Verify installation
node --version
npm --version
```

### Version Management (Optional)
```bash
# Install nvm for multiple Node versions
paru -S nvm
# or: yay -S nvm

# Usage
nvm install node        # latest
nvm install --lts       # latest LTS
nvm use 18             # switch versions
```

## Essential Commands

### Package Management
```bash
# Initialize new project
npm init -y

# Install packages
npm install express     # production dependency
npm install -D nodemon  # dev dependency

# Global install
npm install -g typescript

# Update packages
npm update
npm audit fix          # fix security issues
```

### Running Projects
```bash
# Run directly
node app.js

# Using npm scripts (in package.json)
npm start
npm run dev
npm test
```

## Quick Project Setup

```bash
# Create and setup new project
mkdir my-project && cd my-project
npm init -y
npm install express
echo "console.log('Hello Node!');" > app.js
node app.js
```

## Troubleshooting

### Permission Issues
```bash
# Fix npm global permissions
mkdir ~/.npm-global
npm config set prefix '~/.npm-global'
echo 'export PATH=~/.npm-global/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
```

### Clear Cache
```bash
npm cache clean --force
```

## Common Development Tools
```bash
# Essential dev tools
npm install -g nodemon typescript ts-node
npm install -g @angular/cli create-react-app
```

## VS Code Setup

### Install VS Code
```bash
# Install Visual Studio Code from AUR
yay -S visual-studio-code-bin
```

### Configure User Settings
Create/edit VS Code settings file:
```bash
# Create config directory if needed
mkdir -p ~/.config/Code/User

# Add user settings
cat > ~/.config/Code/User/settings.json << 'EOF'
{
  // workbench
  "workbench.colorTheme": "Tokyo Night",

  // editor
  "editor.fontFamily": "'Monaspace Neon', Consolas, 'Courier New', monospace",
  "editor.fontWeight": "100",
  "editor.fontSize": 14,
  "editor.fontLigatures": true,
  "editor.tokenColorCustomizations": {
    "textMateRules": [
      {
        "scope": ["comment", "emphasis"],
        "settings": { "fontStyle": "italic" }
      },
      {
        "scope": ["storage,entity,variable"],
        "settings": { "fontStyle": "" }
      }
    ]
  },
  "editor.tabSize": 2,
  "editor.insertSpaces": true,
  "editor.detectIndentation": false,

  // ui
  "editor.minimap.enabled": false,
  "editor.scrollbar.vertical": "auto",
  "editor.scrollbar.horizontal": "auto",
  "editor.cursorSmoothCaretAnimation": "on",
  "editor.cursorBlinking": "smooth",
  "window.commandCenter": false,
  "window.zoomLevel": 1,

  // behaviour
  "editor.formatOnSave": true,
  "editor.codeActionsOnSave": {
    "source.organizeImports": "explicit"
  },
  "editor.inlineSuggest.enabled": true,

  // we use SHADCN around here
  "typescript.preferences.autoImportFileExcludePatterns": ["@radix-ui"],

  //git
  "git.confirmSync": false,
  "git.enableSmartCommit": true,

  // formatting
  "[javascript]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode"
  },
  "[typescriptreact]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode"
  },
  "[typescript]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode"
  },
  "[json]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode"
  },
  "[jsonc]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode"
  },
  "[prisma]": {
    "editor.defaultFormatter": "Prisma.prisma"
  },
  "prisma.showPrismaDataPlatformNotification": false,
  "typescript.referencesCodeLens.enabled": true,
  "typescript.updateImportsOnFileMove.enabled": "always",
  "javascript.updateImportsOnFileMove.enabled": "always",
  "[html]": {
    "editor.defaultFormatter": "vscode.html-language-features"
  },

  // AI
  "github.copilot.enable": {
    "*": true,
    "yaml": false,
    "plaintext": false,
    "markdown": true
  },
  "supermaven.enable": {
    "*": false,
    "typescript": true,
    "typescriptreact": true
  },
  "workbench.activityBar.location": "top",
  "workbench.sideBar.location": "right"
}
EOF
```

### Essential Extensions
Install key extensions for your workflow:
```bash
# Install via command line (after VS Code is running)
code --install-extension formulahendry.auto-close-tag
code --install-extension formulahendry.auto-complete-tag
code --install-extension formulahendry.auto-rename-tag
code --install-extension aaron-bond.better-comments
code --install-extension usernamehw.errorlens
code --install-extension dbaeumer.vscode-eslint
code --install-extension eamodio.gitlens
code --install-extension unifiedjs.vscode-mdx
code --install-extension PKief.material-icon-theme
code --install-extension esbenp.prettier-vscode
code --install-extension bradgashler.prettier-sql-vscode
code --install-extension Prisma.prisma
code --install-extension alexcvzz.vscode-sqlite
code --install-extension qwtel.sqlite-viewer
code --install-extension bradlc.vscode-tailwindcss
code --install-extension enkia.tokyo-night
code --install-extension VisualStudioExptTeam.vscodeintellicode
code --install-extension zxh404.vscode-proto3
code --install-extension ZixuanChen.vitest-explorer
```

### Install Monaspace Font
```bash
# Install Monaspace Neon font for ligatures
yay -S ttf-monaspace
```

That's it! Node.js and VS Code are now ready on your Omarchy system.

## Sources

- [VS Code Extensions Reference](https://github.com/KhalidAdan/vscode-stuff)