1. 在命令行中执行 `npm install -g @anthropic-ai/claude-code` 安装 claude code
2. 安装成功后，执行下面脚本，修改设置：
```
node --eval "
    const homeDir = os.homedir();
    const filePath = path.join(homeDir, '.claude.json');
    if (fs.existsSync(filePath)) {
        const content = JSON.parse(fs.readFileSync(filePath, 'utf-8'));
        fs.writeFileSync(filePath, JSON.stringify({ ...content, hasCompletedOnboarding: true }, null, 2), 'utf-8');
    } else {
        fs.writeFileSync(filePath, JSON.stringify({ hasCompletedOnboarding: true }), 'utf-8');
    }"
```
（等价于在 `~/.claude.json` 中添加属性 `"hasCompletedOnboarding": true`）

3. 在vscode中安装ClaudeCode插件
4. 修改vscode settings.json设置，添加：
```
    "claudeCode.environmentVariables": [
        {
            "name": "ANTHROPIC_BASE_URL",
            "value": "https://api.kimi.com/coding/"
        },
        {
            "name": "ANTHROPIC_API_KEY",
            "value": "<your api key>"
        }
    ]
```
5. 在vscode中打开 claude code 进行测试。
