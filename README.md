# JavaScript Google地图抓取：不被封IP的完整采集方案

## 为什么我开始研究Google地图数据采集

上个月接了个本地营销的活儿，客户要我把某个城市所有牙科诊所的名称、地址、电话、评分全部整理成表格。手动复制？三千多条数据，想就头皮发麻。

我第一反应是写个爬虫直接抓Google Maps。结果跑了不到两百条，IP就被封了。换了代理池，质量参差不齐，一半请求超时。折腾了两天，最后靠ScraperAPI才把这事稳定跑通。

**这篇文章写给谁：** 如果你需要用JavaScript批量采集Google地图上的商家数据（POI信息），又不想在反爬机制上浪费时间，这里有一套我验证过的完整流程。核心工具是ScraperAPI，它帮你处理IP轮换、验证码和请求头伪装，你只管写解析逻辑。

---

## Google地图的数据结构长什么样

在动手写代码之前，得先搞清楚Google Maps的数据是怎么组织的。

当你在Google Maps搜索"北京 咖啡馆"，返回的结果列表里每条记录包含：

- 商家名称
- 地址
- 电话号码
- 评分和评价数量
- 营业时间
- 商家类别标签
- 经纬度坐标

这些数据并不是简单的HTML渲染。Google Maps大量使用动态加载和JavaScript渲染，传统的HTTP请求拿到的页面几乎是空壳。这就是为什么普通的`axios + cheerio`方案在这里行不通。

你需要的是能渲染JavaScript的抓取方案。两条路：自己搭Puppeteer/Playwright，或者用ScraperAPI的`render=true`参数让它帮你渲染。

我两种都试过。自己搭浏览器集群，维护成本高，而且Google的反爬检测对无头浏览器特征识别越来越精准。

---

## 用ScraperAPI + JavaScript实现Google地图抓取

### 第一步：拿到API Key

注册ScraperAPI账号后，在Dashboard里能看到你的API Key。免费套餐给5000次API调用，足够你跑通整个流程、验证数据质量。

### 第二步：安装依赖

```bash
npm init -y
npm install axios
```

就这两个。不需要Puppeteer，不需要管理浏览器实例。

### 第三步：构造Google Maps搜索请求

ScraperAPI提供了专门的Google搜索结构化端点，但对于Maps数据，我更倾向用通用端点配合Google Maps的搜索URL：

```javascript
const axios = require('axios');

const API_KEY = '你的ScraperAPI密钥';
const query = '牙科诊所';
const location = '上海';

// Google Maps搜索URL
const targetUrl = `https://www.google.com/maps/search/${encodeURIComponent(query + ' ' + location)}`;

async function scrapeGoogleMaps() {
  try {
    const response = await axios.get('https://api.scraperapi.com', {
      params: {
        api_key: API_KEY,
        url: targetUrl,
        render: 'true',        // 关键：启用JS渲染
        country_code: 'us',    // 可按需切换地区
      },
      timeout: 6000,
    });

    const html = response.data;
    // 接下来解析HTML
    console.log('页面抓取成功，长度:', html.length);
    return html;
  } catch (error) {
    console.error('抓取失败:', error.message);
  }
}

scrapeGoogleMaps();
```

`render: 'true'`这个参数是关键。没有它，你拿到的就是一堆未执行的JavaScript代码。

### 第四步：解析商家数据

Google Maps渲染后的页面结构经常变动，但核心数据通常藏在特定的`div`属性里。用`cheerio`做解析：

```bash
npm install cheerio
```

```javascript
const cheerio = require('cheerio');

function parseBusinessData(html) {
  const $ = cheerio.load(html);
  const results = [];

  // Google Maps的商家卡片通常在带有特定aria-label的容器中
  $('div[role="feed"] > div').each((index, element) => {
    const card = $(element);
    const name = card.find('div.fontHeadlineSmall').text().trim();
    const rating = card.find('span[role="img"]').attr('aria-label') || '';
    const address = card.find('div.fontBodyMedium span').first().text().trim();
    if (name) {
      results.push({
        name,
        rating,
        address,
        // 电话和详细信息需要进入详情页抓取
      });
    }
  });

  return results;
}
```

说实话，选择器会随Google更新而变化。我的做法是每次跑之前先抓一页，肉眼确认结构，再调整选择器。这比写一个"万能解析器"实际得多。

### 第五步：翻页与批量采集

Google Maps的搜索结果是滚动加载的，没有传统分页。但通过ScraperAPI配合不同的搜索偏移参数，可以模拟翻页效果：

```javascript
async function scrapeAllPages(query, location, maxPages = 5) {
  const allResults = [];

  for (let page = 0; page < maxPages; page++) {
    const start = page * 20;
    const targetUrl = `https://www.google.com/maps/search/${encodeURIComponent(query + ' + location)}/@31.2,121.4,12z`;

    const response = await axios.get('https://api.scraperapi.com', {
      params: {
        api_key: API_KEY,
        url: targetUrl,
        render: 'true',},
      timeout: 60000,
    });

    const results = parseBusinessData(response.data);
    allResults.push(...results);

    console.log(`第${page + 1}页抓取完成，累计${allResults.length}条`);

    // 控制请求频率，避免触发限制
    await new Promise(resolve => setTimeout(resolve, 3000));
  }

  return allResults;
}
```

### 第六步：获取商家详情

列表页的信息有限。要拿到电话、营业时间、网站等完整信息，需要进入每个商家的详情页：

```javascript
async function scrapeBusinessDetail(placeUrl) {
  const response = await axios.get('https://api.scraperapi.com', {
    params: {
      api_key: API_KEY,
      url: placeUrl,
      render: 'true',
    },
    timeout: 60000,
  });

  const $ = cheerio.load(response.data);

  return {
    phone: $('button[data-item-id="phone:tel"]').attr('aria-label') || '',
    website: $('a[data-item-id="authority"]').attr('href') || '',
    hours: $('div[aria-label*="时间"], div[aria-label*="hours"]').text().trim(),
  };
}
```

每条详情页消耗一次API调用。如果你要抓三千家商户的完整信息，就是三千多次调用。这时候套餐选择就很重要了。

---

## ScraperAPI套餐怎么选

我跑那个牙科诊所项目，列表页抓了大概150次请求，详情页三千多次，加上调试阶段的消耗，总共用了大约五千次调用。如果你的项目规模类似或更大，免费额度肯定不够。

| 套餐 | API调用额度 | 并发数 | 价格 | 操作 |
|------|----------|--------|------|------|
| Hobby（免费） | 5,000次/月 | 1线程 | $0 | [免费注册开始使用](https://www.scraperapi.com/signup?fp_ref=coupons) |
| Starter | 100,000次/月 | 10线程 | $49/月 | [查看Starter套餐详情](https://www.scraperapi.com/?fp_ref=coupons) |
| Business | 250,000次/月 | 50线程 | $149/月 | [查看Business套餐当前价格](https://www.scraperapi.com/?fp_ref=coupons) |
| Enterprise | 自定义额度 | 自定义并发 | 按需报价 | [联系获取Enterprise方案](https://www.scraperapi.com/?fp_ref=coupons) |

我个人用的是Starter。十个并发线程意味着可以同时跑十个请求，三千条详情页大概半小时就能跑完。如果你只是偶尔抓几百条数据验证想法，免费套餐完全够用。

所有套餐都支持7天退款，不满意可以退。我当时就是先开了一个月Starter试水，确认稳定后才继续续费的。

---

## 我踩过的坑和解决办法

### 请求超时

Google Maps页面渲染本身就慢，加上ScraperAPI需要启动浏览器实例，超时设太短会大量失败。我把timeout从默认的30秒调到60秒，失败率从三成降到不到5%。

### 选择器失效

Google大概每隔几周会调整Maps的DOM结构。我的应对方式是把选择器集中写在一个配置文件里，失效时只改配置不动主逻辑。同时保留原始HTML快照，方便对比排查。

### 数据去重

同一家店可能出现在不同搜索词的结果里。我用商家名称+地址做联合去重键，简单有效。

```javascript
const seen = new Set();

function deduplicate(results) {
  return results.filter(item => {
    const key = `${item.name}_${item.address}`;
    if (seen.has(key)) return false;
    seen.add(key);
    return true;
  });
}
```

### 被Google弹验证码

这是我最初自己搭代理池时的噩梦。换了ScraperAPI之后基本没再遇到过。它内部的IP池和请求指纹管理确实省心，这也是我愿意为它付费的核心原因。

---

## 抓取结果导出

拿到数据后，导出成CSV方便后续处理：

```javascript
const fs = require('fs');

function exportToCSV(data, filename = 'google_maps_data.csv') {
  const headers = Object.keys(data[0]).join(',');
  const rows = data.map(item =>
    Object.values(item).map(v => `"${String(v).replace(/"/g, '""')}"`).join(',')
  );

  const csv = [headers, ...rows].join('\n');
  fs.writeFileSync(filename, csv, 'utf-8');
  console.log(`已导出${data.length}条数据到 ${filename}`);
}
```

---

## 常见问题

### 用JavaScript抓取Google地图数据会被封号吗？

Google Maps没有"账号"概念供你登录抓取，封的是IP。用ScraperAPI的好处就是它帮你轮换IP，你自己的IP永远不会暴露给Google。我跑了几个月，没遇到过任何问题。

### 免费的5000次调用额度能抓多少数据？

取决于你的需求。如果只抓列表页不进详情，一次调用返回约20条结果，5000次理论上能拿10万条基础信息。但如果每条都要进详情页，5000次就只够抓5000家商户的完整数据。建议先用免费额度验证流程，确认可行再升级。

👉[注册免费账号测试你的抓取流程](https://www.scraperapi.com/signup?fp_ref=coupons)

### ScraperAPI的响应速度怎么样？

开启`render=true`后，单次请求通常在8到15秒之间返回。不开渲染的话快很多，两三秒就行。Google Maps必须开渲染，所以批量抓取时并发数就很关键——Starter的10线程和Business的50线程差距在这里体现。

### 除了Google地图还能抓什么？

ScraperAPI不限定目标网站。我还用它抓过Amazon商品页、LinkedIn公司信息、各种电商平台的价格数据。同一个API Key，同一套代码结构，换个URL就行。

---

## 完整代码汇总

把上面的片段整合成一个可直接运行的脚本：

```javascript
const axios = require('axios');
const cheerio = require('cheerio');
const fs = require('fs');

const API_KEY = '你的ScraperAPI密钥';

async function scrapeGoogleMaps(query, location, maxPages = 3) {
  const allResults = [];

  for (let page = 0; page < maxPages; page++) {
    const targetUrl = `https://www.google.com/maps/search/${encodeURIComponent(query + ' + location)}`;

    try {
      const response = await axios.get('https://api.scraperapi.com', {
        params: {
          api_key: API_KEY,
          url: targetUrl,
          render: 'true',
        },
        timeout: 60000,
      });

      const $ = cheerio.load(response.data);

      $('div[role="feed"] > div').each((_, el) => {
        const card = $(el);
        const name = card.find('div.fontHeadlineSmall').text().trim();
        if (name) {
          allResults.push({
            name,
            rating: card.find('span[role="img"]').attr('aria-label') || '',
            address: card.find('div.fontBodyMedium span').first().text().trim(),
          });
        }
      });

      console.log(`第${page + 1}页完成，累计${allResults.length}条`);
    } catch (err) {
      console.error(`第${page + 1}页失败:`, err.message);
    }

    await new Promise(r => setTimeout(r, 3000));
  }

  // 去重
  const seen = new Set();
  const unique = allResults.filter(item => {
    const key = `${item.name}_${item.address}`;
    if (seen.has(key)) return false;
    seen.add(key);
    return true;
  });

  // 导出
  if (unique.length > 0) {
    const headers = Object.keys(unique[0]).join(',');
    const rows = unique.map(item =>
      Object.values(item).map(v => `"${String(v).replace(/"/g, '""')}"`).join(',')
    );
    fs.writeFileSync('maps_data.csv', [headers, ...rows].join('\n'), 'utf-8');
    console.log(`导出${unique.length}条去重数据到 maps_data.csv`);
  }

  return unique;
}

// 运行
scrapeGoogleMaps('咖啡馆', '北京');
```

---

## 最后说两句

Google地图数据采集这件事，技术门槛其实不高。难的是稳定性——IP被封、验证码拦截、页面结构变动，这些才是真正吃时间的地方。

我的建议很简单：把反爬这层交给专门的服务去处理，你的精力放在数据解析和业务逻辑上。ScraperAPI在这个环节帮我省了大量调试时间，尤其是它的IP池质量和JS渲染能力，对Google系产品的抓取确实稳。

如果你的项目刚起步，先用免费额度把流程跑通。确认数据质量符合预期，再根据量级选套餐。

👉[免费注册ScraperAPI，拿5000次调用额度开始你的Google地图抓取项目](https://www.scraperapi.com/signup?fp_ref=coupons)
