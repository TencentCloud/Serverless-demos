## 基于 Serverless 快速定制你所在城市的 nCoV 疫情查询页

> 本文分享了基于 Serverless 实现一个疫情查询页的实现逻辑和部署过程"

![疫情展示图](https://img.serverlesscloud.cn/2020212/1581508115042-%E7%96%AB%E6%83%85.jpg)

每天看到刷屏的疫情人数增长，难免会感到焦虑和恐慌。博主本人身在深圳，最近企业逐步开始复工，更希望可以密切监控自己所在城市的疫情新增情况和趋势，方便掌握信息。

因此借助 Serverless 快速定制了深圳市的疫情数据页面，这里将实现思路和代码分享出来，希望各位也可以基于自己的所在城市方便查看到 新增/累计 的趋势图，更好地对抗疫情。

### 实现效果
![Alt text](https://img.serverlesscloud.cn/2020212/1581480946201-1581479410309.png)
表格部分的实现效果是这样的，可以方便的查看到所在城市近 10 天来的新增、确诊和治愈人数，并且每日更新。

### 实现逻辑

该疫情数据展示页根据 Serverless Framework 构建。代码的改造基于该项目[部署 Serverless 全栈 WEB 应用（Vue.js）](https://github.com/serverless/components/tree/master/templates/tencent-fullstack-vue-application)

- 图表展示部分使用到了 [v-charts](https://v-charts.js.org/#/)
- 数据源的 json 来源如下，建议做好缓存，不要频繁调用。如果希望查询其他城市，将 `city` 字段改为对应的城市名即可。接口文档详情参考[武汉肺炎冠状病毒疫情信息接口 Api
](https://qinhaolei.com/posts/46178/)
`https://myapi.ihogu.com/public/?s=Whfy.city&city=%E6%B7%B1%E5%9C%B3`

首先，在项目目录的 `serverless.yml` 文件中，环境变量填写了上文中的数据源

```
name: tencent-fullstack-vue-application

dashboard:
  component: '@serverless/tencent-website'
  inputs:
    code:
      src: dist
      root: dashboard
      hook: npm run build
    env:
      apiUrl: ${api.url}
      apiUrlSZ: https://myapi.ihogu.com/public/?s=Whfy.city&city=%E6%B7%B1%E5%9C%B3

api:
  component: '@serverless/tencent-express'
  inputs:
    code: ./api
    functionName: tencent-fullstack-vue-api
    apigatewayConf:
      protocols:
        - https
```

之后，在前端页面 `dashboard/index.html` 中，增加了对应的图表展示代码：
```html
        <br>
        <div class="chart-source-data-box">
          <button v-on:click="queryServer">Get Server Time</button><br>
          <div style="color: aliceblue;">{{ message }}</div>
        </div>

        <br>
        <div class="sls-div tagline">
          点击查询深圳市疫情趋势图
        </div>
        <div class="chart-source-data-box">
          <button @click="getPatient">Get Patients</button><br>
        </div>
        <div class="chart-wrapper">
          <div class="chart-no-data" v-if="chartData.rows.length === 0">No Data</div>
          <ve-line class="chart" :data="chartData" :extend="extend" :settings="chartSettings" height="100%" v-if="chartData.rows.length > 0"/>
        </div>
```

其次，在  `dashboard/src/index.js` 中，增加如下逻辑，用于解析疫情的 Json 数据，并且放入表格中。

```javascript
'use strict'

require('../env')

const Vue = require('vue')
const VCharts = require('v-charts')

Vue.use(VCharts)

module.exports = new Vue({
  el: '#root',
  data () {
    this.extend = {
      series: {
        label: { normal: { show: true } }
      }
    },
    this.chartSettings = {
      labelMap:{
        'confirm': '确诊人数',
        'heal': '治愈人数',
        'suspect':'新增人数'
      }
    }
    return {
      message: 'Click me!',
      isVisible: true,
      chartData: {
        columns: ['create_time','confirm','suspect','heal'],
        rows: []
      }
    }
  },
  methods: {
    async queryServer() {
      const response = await fetch(window.env.apiUrl)
      const result = await response.json()
      this.message = result.message
    },
    async getPatient() {
      const response = await fetch(window.env.apiUrlSZ)
      const result = await response.json()
      // handle chart data
      const rows = result.data.items;
      // add new patient
      for (var days = 0; days < rows.length-1; days++){
        rows[days].suspect = rows[days].confirm-rows[days+1].confirm;
      }
      rows.sort((a, b) => {
        return new Date(a.create_time) - new Date(b.create_time);
      })
      this.chartData.rows = rows
    }
  }
})
```

### 快速部署

#### 1. 安装 Serverless 框架和依赖

首先，通过如下命令安装 [Serverless Framework](https://www.github.com/serverless/serverless):

```console
$ npm i -g serverless
```

之后可以新建一个空的文件夹，使用 `create --template-url`，安装相关 template。

```console
$ serverless create --template-url https://github.com/tinafangkunding/nCov-page
```

使用 `cd` 命令，进入 `\nCov-page` 文件夹，可以查看到如下目录结构：

```
|- api
|- dashboard
|- serverless.yml      # 使用项目中的 yml 文件
```

在 `\nCov-page` 目录下，运行如下命令分别安装目录下的 NPM 依赖：
```console
$ npm run bootstrap
```
bootstrap会自动安装目录下的依赖，输出如下所示
```console
> node scripts/bootstrap.js

Start install dependencies...

Root dependencies installed success.
Api dependencies installed success.
Dashboard dependencies installed success.
All dependencies installed.
```

#### 2. 部署到云端

回到 `nCov-page` 目录下，直接通过 `serverless` 命令来部署应用:

```console
$ serverless
```

如果希望查看部署详情，可以通过调试模式的命令 `serverless --debug` 进行部署。

出于成本考虑，这个 web 页面完全搭建在腾讯云云函数 SCF，对象存储 COS 和 API 网关等服务上。在部署阶段，可以直接通过微信扫描命令行中的二维码进行腾讯云的授权登陆和注册。

部署成功后，可以直接在浏览器中访问日志中返回的 dashboard url 地址，查看该疫情查询页的效果:

```bash
  dashboard: 
    url: http://9u9ywac-n56qlg-1251971143.cos-website.ap-guangzhou.myqcloud.com
    env: 
      apiUrl:   https://service-l77nemo2-1251971143.gz.apigw.tencentcs.com/release/
      apiUrlSZ: https://myapi.ihogu.com/public/?s=Whfy.city&city=%E6%B7%B1%E5%9C%B3
  api: 
    region:              ap-guangzhou
    functionName:        tencent-fullstack-vue-api
    apiGatewayServiceId: service-l77nemo2
    url:                 https://service-l77nemo2-1251971143.gz.apigw.tencentcs.com/release/

  21s › dashboard › done
```

该示例基于[腾讯云 Serverless Framework](https://cloud.tencent.com/product/sf) 实现，疫情页本身是静态展示，但本次的模板部署也搭建了 express.js 的后端框架，并且简单实现了从前端页面获取后端服务器时间 Server Time 的逻辑，因此可以方便地扩展更多功能。本文亦发布于[知乎](https://www.zhihu.com/question/370049229/answer/1012268374)

**源码：** [nCov-page 项目](https://github.com/tinafangkunding/nCov-page)。

> **传送门：**
>
> - GitHub: [github.com/serverless](https://github.com/serverless/serverless/blob/master/README_CN.md) 
> - 官网：[serverless.com](https://serverless.com/)

欢迎访问：[Serverless 中文网](https://china.serverless.com)，您可以在 [最佳实践](https://china.serverless.com/best-practice/) 里体验更多关于 Serverless 应用的开发！