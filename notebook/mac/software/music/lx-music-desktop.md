升级安装：

https://github.com/lyswhut/lx-music-desktop

```shell
cd /usr/local/workspace/github/web/lx-music-desktop
rm -R lx-music-desktop/
gh repo clone lyswhut/lx-music-desktop
cd lx-music-desktop
#npm install -g cross-env
#npm config set electron_mirror https://npm.taobao.org/mirrors/electron/
npm install chalk
npm run pack:dir

```

