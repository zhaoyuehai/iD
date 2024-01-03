# iD编辑器

## 一、配置

1. 新建.env文件，参考config/envs.mjs和config/id.js进行配置
    ```
    ID_API_CONNECTION_URL=http://192.168.x.x:x
    ID_API_CONNECTION_API_URL=http://192.168.x.x:x
    ID_API_CONNECTION_CLIENT_ID=xxxx
    ID_API_CONNECTION_CLIENT_SECRET=xxx
    ID_CUSTOM_PRESETS_URL=http://192.168.x.x:x/preset_menu/
    ```

2. 修改/config/envs.mjs和id.js文件，提供环境配置：`customPresetsUrl`和`apiUrl: ENV__ID_API_CONNECTION_API_URL`
3. 修改/modules/core/file_fetcher.js和localizer.js文件，替换配置`presetsCdnUrl`为对应的`customPresetsUrl`。
4. 修改/scripts/build_data.js文件，替换配置`presetsUrl`为对应的`customPresetsUrl`。
    ```
    const customPresetsUrl = process.env.ID_CUSTOM_PRESETS_URL;//TODO
    ...
     Promise.all([
          // Fetch the icons that are needed by the expected tagging schema version
          // fetchOrRequire(`${presetsUrl}/dist/presets.min.json`),
          fetchOrRequire(`${customPresetsUrl}presets.json`),//TODO
          // fetchOrRequire(`${presetsUrl}/dist/preset_categories.min.json`),
          fetchOrRequire(`${customPresetsUrl}preset_categories.json`),//TODO
          // fetchOrRequire(`${presetsUrl}/dist/fields.min.json`),
          fetchOrRequire(`${customPresetsUrl}fields.json`),//TODO
    ...
    ```

## 二、运行

注意当前的node版本，如果版本太低运行npm install会报错

```
npm install
npm run all #构建
npm start #启动
npm run dist #打包
```

访问：http://localhost:8080

### 异常情况1：

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

### 异常情况2：

ENV__ID_API_CONNECTION_API_URL is not defined

解决：
参考上面**调整修改**第1、2条。添加对应的ENV变量`ID_API_CONNECTION_API_URL=http://192.168.x.x:x`

## 三、docker

`docker build -t openstreetmap-editor:v1.0.0 .`

`docker run -d -p 8080:80 --name osm-editor openstreetmap-editor:v1.0.0`

## 四、修改成Basic认证

注意：**osm后端支持Basic auth认证**。当前前端项目使用osm-auth库`"osm-auth": "~2.3.0"`
来实现编辑器的登录认证功能。osm-auth默认使用的是oauth2认证方式，现将其修改为basic认证方式。

1. 复制需要的osm-auth库文件到/modules/custom/osm-auth目录下。
2. 调整osm.js文件：将/modules/services/osm.js中导入的osmAuth库替换成自己的。
   ```js
    // import { osmAuth } from 'osm-auth';
   import {osmAuth} from '../custom/osm-auth/src/osm-auth.mjs';
   ```
3. 修改调整/modules/custom/osm-auth/src/osm-auth.mjs
   ```js
   function _doFetch() {
       options = options || {};
       if (!options.headers) {
           options.headers = {'Content-Type': 'application/x-www-form-urlencoded'};
       }
       // options.headers.Authorization = 'Bearer ' + token('oauth2_access_token');
       options.headers.Authorization = 'Basic ' + token('basic_auth_token');
       return fetch(resource, options);
   }
   ```
   ```
   ...

   function _doXHR() {
       var url = options.prefix !== false ? (o.apiUrl + options.path) : options.path;
         return oauth.rawxhr(
           options.method,
           url,
           // `Bearer ${token('oauth2_access_token')}`,
           `Basic ${token('basic_auth_token')}`,
           options.content,
           options.headers,
           done
         );
   }

   oauth.rawxhr = function(method, url, authorization, data, headers, callback) {
       headers = headers || { 'Content-Type': 'application/x-www-form-urlencoded' };
       if (authorization) {
         headers.Authorization = authorization
       }

       var xhr = new XMLHttpRequest();
   ...

   ```

## 五、雷达点云图

在/modules/renderer/background.js中的`export function rendererBackground(context){ ... }`方法中修改添加自定义的雷达点云（Radar
Point Cloud）图层：

```
function ensureImageryIndex() {
  return fileFetcher.get('imagery')
    .then(sources => {

      ...


      //添加雷达点云图 TODO
      let radar = rendererBackgroundSource.Radar();
      _imageryIndex.backgrounds.unshift(radar);
      return _imageryIndex;
    });
}
```

在/modules/renderer/background_source.js中提供`rendererBackgroundSource.Radar()`：

```
//radar point cloud 雷达点云图 TODO
rendererBackgroundSource.Radar = function (){
    var id = 'radar_point_cloud';

    var source = rendererBackgroundSource({ id: id, template: radarBackgroundTemplate});

    source.name = function() {
        return t(`background.${id}`);
    };

    //注意需要为此处进行多言适配
    source.label = function() {
        return t.append(`background.${id}`);
    };


    source.imageryUsed = function() {

    ...

}
```

在dist/locales目录下的zh-CN.min.json等文件配置多语言："radar_point_cloud":"雷达点云图"

` ... "background":{"title":"背景","description":"背景设定", ... "custom":"自定义","radar_point_cloud":"雷达点云图","overlays":"叠加图层" ... `
