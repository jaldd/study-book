### vscode报错：gopls requires a module at the root of your workspace.



##### 第1步：打开 Vscode，然后转到设置。

##### 第2步：在搜索栏中，输入 gopls

##### 第3步：在它的下方，找到 settings.json，点击它来编辑

##### 第4步：在代码部分下面增加

```json
"gopls": {
    "experimentalWorkspaceModule": true,
}
```

