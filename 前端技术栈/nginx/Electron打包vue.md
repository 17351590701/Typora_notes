1. 安装Electron依赖

   ```bash
   npm install electron --save-dev
   ```

2. 项目根目录创建 **Electron入口文件** `main.js`

   ```js
   const { app, BrowserWindow } = require('electron');
   const path = require('path');
   
   function createWindow() {
     // 创建浏览器窗口
     let win = new BrowserWindow({
       width: 800,
       height: 600,
       webPreferences: {
         nodeIntegration: true
       }
     });
   
     // 并且为你的应用加载index.html
     win.loadFile(path.join(__dirname, 'dist/index.html'));
   }
   
   // 当Electron完成初始化并且准备创建浏览器窗口时，会调用这个方法
   app.whenReady().then(createWindow);
   
   // 关闭所有窗口时退出应用
   app.on('window-all-closed', () => {
     if (process.platform !== 'darwin') {
       app.quit();
     }
   });
   
   app.on('activate', () => {
     if (BrowserWindow.getAllWindows().length === 0) {
       createWindow();
     }
   });
   ```

3. 配置Electron打包工具，安装`electron-builder`

   ```bash
   npm install electron-builder --save-dev
   ```

4. 在`package.json`中配置`electron-builder`