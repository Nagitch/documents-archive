## 結果

<p class="codepen" data-height="265" data-theme-id="light" data-default-tab="js,result" data-user="nagitch" data-slug-hash="rNMrXNw" style="height: 265px; box-sizing: border-box; display: flex; align-items: center; justify-content: center; border: 2px solid; margin: 1em 0; padding: 1em;" data-pen-title="C3js TimeSeries Realtime Graph">
  <span>See the Pen <a href="https://codepen.io/nagitch/pen/rNMrXNw">
  C3js TimeSeries Realtime Graph</a> by Nagitch (<a href="https://codepen.io/nagitch">@nagitch</a>)
  on <a href="https://codepen.io">CodePen</a>.</span>
</p>
<script async src="https://production-assets.codepen.io/assets/embed/ei.js"></script>


## 解説

### 過去1分間の時間軸を表示する

軸の設定をtype: timeseries(時系列), max/minに現在秒, 一分前の秒にしておくと自動的に過去1分間の時間軸を表示してくれます。
一秒刻みでデータを登録して無理やりプロットする、みたいなトリックは使わなくて大丈夫です。
また `2021-01-20 19:47:35` のような表示にする＆1分前の計算のためにdayjsを使っていますが、同じ形になれば何を使っても問題ありません。

```js

const timeNow = () => dayjs().format('YYYY-MM-DD HH:mm:ss');
const timeTail = () => dayjs().subtract(1, 'm').format('YYYY-MM-DD HH:mm:ss');

const chartAxis = {
  x: {
    type: 'timeseries',
    min: timeTail(),
    max: timeNow(),
```

また等間隔に満遍なく軸表示するために `fit: true`, 表示を倒して見やすくするため `rotate:-50` を指定します。

```js
    tick: {
      fit: true,
      rotate: -50,
      format: '%Y-%m-%d %H:%M:%S',
    }
```

### 時間軸(表示範囲)の更新

時間軸の更新には `axis.min()`, `axis.max()` 関数を使います。直接値を書き換えるだけだと反映されません。 `load()` を叩いても更新されません。

```js
setInterval(() => {
  chart.axis.min({x: timeTail()});
  chart.axis.max({x: timeNow()});
  ...
}, 1000)
```

サンプルでは毎秒新しいデータをプロットしていますが、歯抜けになっても大丈夫です。
好きなタイミングでどの秒にデータを入れても正しくプロットされます。

```js

  chartData.columns[0].push(timeNow());
  chartData.columns[1].push(Math.random());
  chart.load({columns: chartData.columns});
```
