# iD编辑器注意事项

## 1.修改和调整
1. 新建.env文件，参考config/envs.mjs和config/id.js进行配置
    ```
    ID_API_CONNECTION_URL=http://192.168.x.x:x
    ID_API_CONNECTION_API_URL=http://192.168.x.x:x
    ID_API_CONNECTION_CLIENT_ID=xxxx
    ID_API_CONNECTION_CLIENT_SECRET=xxx
    ID_PRESETS_CUSTOM_URL=http://192.168.x.x:x/preset_menu/
    ```

2. 修改/config/envs.mjs和id.js文件，提供环境配置：`presetsCustomUrl`和`apiUrl: ENV__ID_API_CONNECTION_API_URL`
3. 修改/modules/core/file_fetcher.js和localizer.js文件，替换配置`presetsCdnUrl`为对应的`presetsCustomUrl`。
4. 修改/scripts/build_data.js文件，替换配置`presetsUrl`为对应的`presetsCustomUrl`。
    ```
    const presetsCustomUrl = process.env.ID_PRESETS_CUSTOM_URL;//TODO
    ...
     Promise.all([
          // Fetch the icons that are needed by the expected tagging schema version
          // fetchOrRequire(`${presetsUrl}/dist/presets.min.json`),
          fetchOrRequire(`${presetsCustomUrl}presets.json`),//TODO
          // fetchOrRequire(`${presetsUrl}/dist/preset_categories.min.json`),
          fetchOrRequire(`${presetsCustomUrl}preset_categories.json`),//TODO
          // fetchOrRequire(`${presetsUrl}/dist/fields.min.json`),
          fetchOrRequire(`${presetsCustomUrl}fields.json`),//TODO
    ...
    ```

5. Run（注意当前的node版本，如果版本太低运行npm install会报错）
    ```
    npm install
    npm run all
    npm start
    ```

### 报错1：
FetchError: request to https://raw.githubusercontent.com/openstreetmap/id-tagging-schema/main/dist/presets.min.json
failed, reason: connect ETIMEDOUT 185.199.110.133:443

解决1：
问题点——> scripts/build_data.js中的Fetch the icons ——> presetsUrl
新建文件夹 preset/dist，下载相关json文件到该文件夹下
修改scripts/build_data.js中的代码：
```
  ...
  Promise.all([
        //...
        JSON.parse(fs.readFileSync('preset/dist/presets.min.json')),
        JSON.parse(fs.readFileSync('preset/dist/preset_categories.min.json')),
        JSON.parse(fs.readFileSync('preset/dist/fields.min.json'))
    ])
    // .then(responses => Promise.all(responses.map(response => response.json())))
    .then((results) => {
      // compile the icons used by all the presets
  ...
```
解决2： 直接访问自己的服务器获取json文件，参考上面**调整修改**第4条

### 报错2：
ENV__ID_API_CONNECTION_API_URL is not defined

解决：
参考上面**调整修改**第1、2条。添加对应的ENV变量`ID_API_CONNECTION_API_URL=http://192.168.x.x:x`

## 2.运行
构建：`npm run all`

启动：`npm start`

打包：`npm run dist`

访问：http://localhost:8080

## 3.docker
`docker build -t openstreetmap-editor:v1.0.0 .`

`docker run -d -p 8080:80 --name osm-editor openstreetmap-editor:v1.0.0`