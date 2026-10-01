# Hacker News event-discourse collection

**Observed window:** September 8-30, 2026. Uses the public Hacker News Algolia search index for stories and comments.

This is a query-defined source sample, not a census of social-media sentiment. It is intentionally separate from the AI-equity event-study notebook.


```python
from pathlib import Path
import html, json, re
import numpy as np
import pandas as pd
import requests
from vaderSentiment.vaderSentiment import SentimentIntensityAnalyzer

START, END = '2026-09-08', '2026-10-01'  # END is exclusive; includes Sep. 30
RAW = Path('data/raw'); RAW.mkdir(parents=True, exist_ok=True)
analyzer = SentimentIntensityAnalyzer()
NEG = re.compile(r'\b(risk|risks|danger|dangerous|warning|warns|kill|killing|death|doom|catastroph|existential|threat|fear|panic|alarm|wipe out|humanity)\b', re.I)
POS = re.compile(r'\b(safe|safety|benefit|opportunity|growth|progress|solution|guardrail|optimism|improve)\b', re.I)

def score_daily(posts):
    if posts.empty:
        return pd.DataFrame()
    posts = posts.copy()
    posts['date'] = pd.to_datetime(posts['date'], utc=True).dt.tz_localize(None).dt.normalize()
    posts = posts.loc[(posts.date >= pd.Timestamp(START)) & (posts.date < pd.Timestamp(END))].drop_duplicates('id')
    posts['compound'] = posts.text.map(lambda x: analyzer.polarity_scores(str(x))['compound'])
    posts['risk_intensity'] = posts.text.map(lambda x: (len(NEG.findall(str(x))) - len(POS.findall(str(x)))) / max(1, len(str(x).split())))
    posts['engagement'] = 1 + posts.likes + posts.replies + posts.quotes + posts.reposts
    return posts, posts.groupby('date').agg(posts=('id', 'size'), compound=('compound', 'mean'), risk_intensity=('risk_intensity', 'mean'))

```


```python
CACHE = RAW / 'hackernews_2026-09-08_2026-10-01.json'
URL = 'https://hn.algolia.com/api/v1/search_by_date'
QUERIES = ['Evan Hubinger', 'Jacob Coxon', 'Anthropic AI risk', 'Anthropic alignment']

def collect_hackernews():
    if CACHE.exists():
        return json.loads(CACHE.read_text())
    rows, errors = [], []
    lower, upper = int(pd.Timestamp(f'{START}T00:00:00Z').timestamp()), int(pd.Timestamp(f'{END}T00:00:00Z').timestamp())
    for query in QUERIES:
        try:
            r = requests.get(URL, params={'query':query, 'tags':'(story,comment)', 'hitsPerPage':100, 'numericFilters':f'created_at_i>={lower},created_at_i<{upper}'}, headers={'User-Agent':'academic-event-study/1.0'}, timeout=30)
            r.raise_for_status()
            for hit in r.json().get('hits', []):
                text = html.unescape(re.sub(r'<[^>]+>', ' ', hit.get('comment_text') or hit.get('story_title') or hit.get('title') or ''))
                item_id = hit.get('objectID')
                rows.append({'id':f'hn-{item_id}', 'date':hit.get('created_at'), 'text':text, 'author':hit.get('author',''), 'url':f'https://news.ycombinator.com/item?id={item_id}', 'likes':hit.get('points') or 0, 'replies':hit.get('num_comments') or 0, 'quotes':0, 'reposts':0})
        except requests.RequestException as exc:
            errors.append(f'{query}: {exc}')
    payload = {'metadata':{'queries':QUERIES, 'start':START, 'end_exclusive':END, 'retrieved_utc':pd.Timestamp.now(tz='UTC').isoformat(), 'errors':errors}, 'rows':rows}
    CACHE.write_text(json.dumps(payload, ensure_ascii=False, indent=2))
    return payload

payload = collect_hackernews()
posts = pd.DataFrame(payload['rows'])
if posts.empty:
    print('No Hacker News observations collected:', payload['metadata'].get('errors', []))
else:
    posts, daily = score_daily(posts)
    print(payload['metadata']); display(daily)

```

    {'queries': ['Evan Hubinger', 'Jacob Coxon', 'Anthropic AI risk', 'Anthropic alignment'], 'start': '2026-09-08', 'end_exclusive': '2026-10-01', 'retrieved_utc': '2026-09-30T21:33:53.739334+00:00', 'errors': []}



<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>posts</th>
      <th>compound</th>
      <th>risk_intensity</th>
    </tr>
    <tr>
      <th>date</th>
      <th></th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>2026-09-08</th>
      <td>3</td>
      <td>0.197267</td>
      <td>0.003584</td>
    </tr>
    <tr>
      <th>2026-09-09</th>
      <td>25</td>
      <td>-0.008688</td>
      <td>0.020589</td>
    </tr>
    <tr>
      <th>2026-09-10</th>
      <td>18</td>
      <td>-0.279739</td>
      <td>0.020060</td>
    </tr>
    <tr>
      <th>2026-09-11</th>
      <td>7</td>
      <td>-0.059400</td>
      <td>0.031867</td>
    </tr>
    <tr>
      <th>2026-09-12</th>
      <td>29</td>
      <td>0.133817</td>
      <td>0.004542</td>
    </tr>
    <tr>
      <th>2026-09-13</th>
      <td>14</td>
      <td>-0.179571</td>
      <td>0.006130</td>
    </tr>
    <tr>
      <th>2026-09-14</th>
      <td>8</td>
      <td>-0.380213</td>
      <td>0.027597</td>
    </tr>
    <tr>
      <th>2026-09-15</th>
      <td>4</td>
      <td>0.430175</td>
      <td>0.006579</td>
    </tr>
    <tr>
      <th>2026-09-16</th>
      <td>9</td>
      <td>-0.109289</td>
      <td>0.006838</td>
    </tr>
    <tr>
      <th>2026-09-17</th>
      <td>7</td>
      <td>-0.147586</td>
      <td>-0.004243</td>
    </tr>
    <tr>
      <th>2026-09-18</th>
      <td>9</td>
      <td>-0.054856</td>
      <td>0.016926</td>
    </tr>
    <tr>
      <th>2026-09-19</th>
      <td>2</td>
      <td>0.184600</td>
      <td>0.000000</td>
    </tr>
    <tr>
      <th>2026-09-20</th>
      <td>6</td>
      <td>0.147500</td>
      <td>0.004789</td>
    </tr>
    <tr>
      <th>2026-09-21</th>
      <td>6</td>
      <td>0.161733</td>
      <td>-0.001327</td>
    </tr>
    <tr>
      <th>2026-09-22</th>
      <td>3</td>
      <td>0.826833</td>
      <td>-0.003633</td>
    </tr>
    <tr>
      <th>2026-09-23</th>
      <td>5</td>
      <td>0.169740</td>
      <td>-0.001845</td>
    </tr>
    <tr>
      <th>2026-09-24</th>
      <td>3</td>
      <td>0.013500</td>
      <td>0.066667</td>
    </tr>
    <tr>
      <th>2026-09-25</th>
      <td>18</td>
      <td>-0.174950</td>
      <td>0.021251</td>
    </tr>
    <tr>
      <th>2026-09-26</th>
      <td>4</td>
      <td>-0.034950</td>
      <td>0.002098</td>
    </tr>
    <tr>
      <th>2026-09-27</th>
      <td>4</td>
      <td>-0.698525</td>
      <td>0.007525</td>
    </tr>
    <tr>
      <th>2026-09-28</th>
      <td>8</td>
      <td>0.134587</td>
      <td>0.003143</td>
    </tr>
    <tr>
      <th>2026-09-29</th>
      <td>12</td>
      <td>-0.425242</td>
      <td>0.083251</td>
    </tr>
    <tr>
      <th>2026-09-30</th>
      <td>7</td>
      <td>0.667529</td>
      <td>0.005442</td>
    </tr>
  </tbody>
</table>
</div>


## Source-level event-study analysis

The analysis below is conditional on returned observations. It uses the same equal-weight AI basket as the original study: NVDA, META, AMZN, GOOGL, MSFT, AVGO, and TSM, benchmarked against SPY. The one-lag Granger diagnostics are exploratory: source coverage is query-defined and the event window is very short.



```python
import warnings
import matplotlib.pyplot as plt
import yfinance as yf
from statsmodels.tsa.stattools import grangercausalitytests

if posts.empty:
    print('Analysis skipped because this source returned no observations.')
else:
    # Add an engagement-weighted score as sensitivity only; the unweighted score is primary.
    weighted = posts.groupby('date').apply(
        lambda d: np.average(d['compound'], weights=d['engagement'])
    ).rename('engagement_weighted_compound')
    daily_analysis = daily.join(weighted)

    TICKERS = ['NVDA', 'META', 'AMZN', 'GOOGL', 'MSFT', 'AVGO', 'TSM']
    closes = yf.download(TICKERS + ['SPY'], start=START, end=END,
                         auto_adjust=True, progress=False)['Close']
    returns = np.log(closes).diff().dropna()
    returns['basket_return'] = returns[TICKERS].mean(axis=1)
    returns['basket_excess_return'] = returns['basket_return'] - returns['SPY']

    panel = pd.DataFrame(index=returns.index).join(daily_analysis).fillna({
        'posts': 0, 'compound': 0, 'risk_intensity': 0,
        'engagement_weighted_compound': 0,
    }).join(returns[['basket_return', 'basket_excess_return']])

    summary = pd.Series({
        'unique posts': len(posts),
        'covered calendar days': int((daily_analysis['posts'] > 0).sum()),
        'mean unweighted compound': posts['compound'].mean(),
        'mean risk intensity': posts['risk_intensity'].mean(),
        'median engagement': posts['engagement'].median(),
        'basket cumulative return': np.exp(returns['basket_return'].sum()) - 1,
        'SPY cumulative return': np.exp(returns['SPY'].sum()) - 1,
    })
    display(summary.to_frame('value'))
    display(posts[['date', 'author', 'text', 'url', 'engagement', 'compound', 'risk_intensity']].sort_values('date').head(20))
    display(panel.round(4))

    fig, ax = plt.subplots(3, 1, figsize=(11, 9), sharex=True)
    ax[0].bar(panel.index, panel['posts'], color='#5b7db1')
    ax[0].set_ylabel('posts')
    ax[0].set_title('Query-defined public-source activity')
    ax[1].plot(panel.index, panel['compound'], marker='o', color='#7b2d8e', label='unweighted compound')
    ax[1].plot(panel.index, panel['engagement_weighted_compound'], marker='x', linestyle='--', color='#e08214', label='engagement-weighted')
    ax[1].axhline(0, color='black', linewidth=.7); ax[1].legend(); ax[1].set_ylabel('sentiment')
    ax[2].plot(panel.index, 100 * panel['basket_excess_return'], marker='o', color='#174a7e')
    ax[2].axhline(0, color='black', linewidth=.7); ax[2].set_ylabel('basket excess log return (%)')
    fig.autofmt_xdate(); plt.tight_layout(); plt.show()

    def granger_pvalue(driver):
        sample = panel[['basket_excess_return', driver]].dropna()
        if len(sample) < 8 or sample[driver].nunique() < 2:
            return np.nan
        with warnings.catch_warnings():
            warnings.simplefilter('ignore')
            result = grangercausalitytests(sample, maxlag=1)
        return result[1][0]['ssr_ftest'][1]

    granger = pd.Series({
        'compound -> excess return': granger_pvalue('compound'),
        'risk intensity -> excess return': granger_pvalue('risk_intensity'),
        'post count -> excess return': granger_pvalue('posts'),
        'weighted compound -> excess return': granger_pvalue('engagement_weighted_compound'),
    }, name='lag-1 F-test p-value')
    display(granger.to_frame())
    print('Interpretation: p-values are descriptive low-power diagnostics, not causal estimates. A zero on a no-post trading day means no observed sample activity, not neutral public sentiment.')

```


<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>value</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>unique posts</th>
      <td>211.000000</td>
    </tr>
    <tr>
      <th>covered calendar days</th>
      <td>23.000000</td>
    </tr>
    <tr>
      <th>mean unweighted compound</th>
      <td>-0.034992</td>
    </tr>
    <tr>
      <th>mean risk intensity</th>
      <td>0.016314</td>
    </tr>
    <tr>
      <th>median engagement</th>
      <td>1.000000</td>
    </tr>
    <tr>
      <th>basket cumulative return</th>
      <td>0.028835</td>
    </tr>
    <tr>
      <th>SPY cumulative return</th>
      <td>-0.001875</td>
    </tr>
  </tbody>
</table>
</div>



<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>date</th>
      <th>author</th>
      <th>text</th>
      <th>url</th>
      <th>engagement</th>
      <th>compound</th>
      <th>risk_intensity</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>22</th>
      <td>2026-09-08</td>
      <td>frgturpwd</td>
      <td>I think this easily could have been me if I di...</td>
      <td>https://news.ycombinator.com/item?id=49608555</td>
      <td>1</td>
      <td>0.8068</td>
      <td>0.000000</td>
    </tr>
    <tr>
      <th>23</th>
      <td>2026-09-08</td>
      <td>throw-qqqqq</td>
      <td>There is huge logistics involved in killing a ...</td>
      <td>https://news.ycombinator.com/item?id=49607750</td>
      <td>1</td>
      <td>-0.8519</td>
      <td>0.010753</td>
    </tr>
    <tr>
      <th>21</th>
      <td>2026-09-08</td>
      <td>Bluestein</td>
      <td>Hey, maybe the scariest part of this is that, ...</td>
      <td>https://news.ycombinator.com/item?id=49614982</td>
      <td>1</td>
      <td>0.6369</td>
      <td>0.000000</td>
    </tr>
    <tr>
      <th>220</th>
      <td>2026-09-09</td>
      <td>nothrowaways</td>
      <td>More than 10% probability AI will wipe out hum...</td>
      <td>https://news.ycombinator.com/item?id=49621712</td>
      <td>5</td>
      <td>0.0000</td>
      <td>0.166667</td>
    </tr>
    <tr>
      <th>151</th>
      <td>2026-09-09</td>
      <td>slowin</td>
      <td>&gt;&nbsp;&nbsp;“I do agree that the public has a negative ...</td>
      <td>https://news.ycombinator.com/item?id=49629327</td>
      <td>1</td>
      <td>-0.7747</td>
      <td>0.019802</td>
    </tr>
    <tr>
      <th>152</th>
      <td>2026-09-09</td>
      <td>lenerdenator</td>
      <td>The risks of AI, while mostly hypothetical, ha...</td>
      <td>https://news.ycombinator.com/item?id=49628337</td>
      <td>1</td>
      <td>-0.5023</td>
      <td>0.020619</td>
    </tr>
    <tr>
      <th>153</th>
      <td>2026-09-09</td>
      <td>barnabee</td>
      <td>This Already powerful and wealthy organisation...</td>
      <td>https://news.ycombinator.com/item?id=49628172</td>
      <td>1</td>
      <td>-0.3491</td>
      <td>0.025000</td>
    </tr>
    <tr>
      <th>154</th>
      <td>2026-09-09</td>
      <td>thiago_fm</td>
      <td>How about we instead make Earth be a planet go...</td>
      <td>https://news.ycombinator.com/item?id=49627829</td>
      <td>1</td>
      <td>-0.9337</td>
      <td>0.000000</td>
    </tr>
    <tr>
      <th>155</th>
      <td>2026-09-09</td>
      <td>dgellow</td>
      <td>&gt; Like any economic model, this one has limits...</td>
      <td>https://news.ycombinator.com/item?id=49627154</td>
      <td>1</td>
      <td>0.5864</td>
      <td>0.022472</td>
    </tr>
    <tr>
      <th>156</th>
      <td>2026-09-09</td>
      <td>shafyy</td>
      <td>Not saying that the current civilizations will...</td>
      <td>https://news.ycombinator.com/item?id=49625397</td>
      <td>1</td>
      <td>-0.8807</td>
      <td>0.014493</td>
    </tr>
    <tr>
      <th>157</th>
      <td>2026-09-09</td>
      <td>lemoncookiechip</td>
      <td>This right here, or at least the thought of th...</td>
      <td>https://news.ycombinator.com/item?id=49624837</td>
      <td>1</td>
      <td>0.9699</td>
      <td>0.004292</td>
    </tr>
    <tr>
      <th>159</th>
      <td>2026-09-09</td>
      <td>andai</td>
      <td>&gt; A common response is “if they truly believe ...</td>
      <td>https://news.ycombinator.com/item?id=49621737</td>
      <td>1</td>
      <td>-0.3658</td>
      <td>0.005319</td>
    </tr>
    <tr>
      <th>160</th>
      <td>2026-09-09</td>
      <td>nvdc</td>
      <td>i don't think everything that comes out like t...</td>
      <td>https://news.ycombinator.com/item?id=49621559</td>
      <td>1</td>
      <td>0.9605</td>
      <td>0.009259</td>
    </tr>
    <tr>
      <th>60</th>
      <td>2026-09-09</td>
      <td>robtherobber</td>
      <td>'Gambling with our lives': AI researcher quits...</td>
      <td>https://news.ycombinator.com/item?id=49623044</td>
      <td>5</td>
      <td>0.0000</td>
      <td>0.000000</td>
    </tr>
    <tr>
      <th>59</th>
      <td>2026-09-09</td>
      <td>taubek</td>
      <td>Gambling with our lives: AI researcher quits A...</td>
      <td>https://news.ycombinator.com/item?id=49623306</td>
      <td>185</td>
      <td>0.1027</td>
      <td>0.000000</td>
    </tr>
    <tr>
      <th>57</th>
      <td>2026-09-09</td>
      <td>wrongful1520</td>
      <td>I resigned from Anthropic today (Jacob Coxon)</td>
      <td>https://news.ycombinator.com/item?id=49624157</td>
      <td>85</td>
      <td>-0.2500</td>
      <td>0.000000</td>
    </tr>
    <tr>
      <th>56</th>
      <td>2026-09-09</td>
      <td>hodder</td>
      <td>Jacob Coxon resignation appears to be a PR stu...</td>
      <td>https://news.ycombinator.com/item?id=49633440</td>
      <td>34</td>
      <td>-0.2960</td>
      <td>0.000000</td>
    </tr>
    <tr>
      <th>55</th>
      <td>2026-09-09</td>
      <td>giardini</td>
      <td>But see also "Jacob Coxon resignation appears ...</td>
      <td>https://news.ycombinator.com/item?id=49633883</td>
      <td>1</td>
      <td>-0.4215</td>
      <td>0.000000</td>
    </tr>
    <tr>
      <th>158</th>
      <td>2026-09-09</td>
      <td>jampekka</td>
      <td>Sad that there's the obvious regulatory captur...</td>
      <td>https://news.ycombinator.com/item?id=49624830</td>
      <td>1</td>
      <td>-0.6209</td>
      <td>0.016667</td>
    </tr>
    <tr>
      <th>20</th>
      <td>2026-09-09</td>
      <td>koolba</td>
      <td>&gt; Evan Hubinger, Anthropic's staff lead on kee...</td>
      <td>https://news.ycombinator.com/item?id=49623733</td>
      <td>1</td>
      <td>-0.8126</td>
      <td>0.018692</td>
    </tr>
  </tbody>
</table>
</div>



<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>posts</th>
      <th>compound</th>
      <th>risk_intensity</th>
      <th>engagement_weighted_compound</th>
      <th>basket_return</th>
      <th>basket_excess_return</th>
    </tr>
    <tr>
      <th>Date</th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>2026-09-09</th>
      <td>25</td>
      <td>-0.0087</td>
      <td>0.0206</td>
      <td>-0.0353</td>
      <td>-0.0016</td>
      <td>0.0031</td>
    </tr>
    <tr>
      <th>2026-09-10</th>
      <td>18</td>
      <td>-0.2797</td>
      <td>0.0201</td>
      <td>-0.0827</td>
      <td>-0.0084</td>
      <td>-0.0024</td>
    </tr>
    <tr>
      <th>2026-09-11</th>
      <td>7</td>
      <td>-0.0594</td>
      <td>0.0319</td>
      <td>-0.3111</td>
      <td>0.0091</td>
      <td>0.0006</td>
    </tr>
    <tr>
      <th>2026-09-14</th>
      <td>8</td>
      <td>-0.3802</td>
      <td>0.0276</td>
      <td>-0.2792</td>
      <td>-0.0077</td>
      <td>-0.0032</td>
    </tr>
    <tr>
      <th>2026-09-15</th>
      <td>4</td>
      <td>0.4302</td>
      <td>0.0066</td>
      <td>0.4909</td>
      <td>-0.0090</td>
      <td>-0.0044</td>
    </tr>
    <tr>
      <th>2026-09-16</th>
      <td>9</td>
      <td>-0.1093</td>
      <td>0.0068</td>
      <td>-0.1824</td>
      <td>-0.0006</td>
      <td>0.0038</td>
    </tr>
    <tr>
      <th>2026-09-17</th>
      <td>7</td>
      <td>-0.1476</td>
      <td>-0.0042</td>
      <td>-0.1476</td>
      <td>0.0200</td>
      <td>0.0087</td>
    </tr>
    <tr>
      <th>2026-09-18</th>
      <td>9</td>
      <td>-0.0549</td>
      <td>0.0169</td>
      <td>-0.0549</td>
      <td>0.0052</td>
      <td>0.0039</td>
    </tr>
    <tr>
      <th>2026-09-21</th>
      <td>6</td>
      <td>0.1617</td>
      <td>-0.0013</td>
      <td>0.1617</td>
      <td>0.0315</td>
      <td>0.0161</td>
    </tr>
    <tr>
      <th>2026-09-22</th>
      <td>3</td>
      <td>0.8268</td>
      <td>-0.0036</td>
      <td>0.8268</td>
      <td>-0.0015</td>
      <td>-0.0014</td>
    </tr>
    <tr>
      <th>2026-09-23</th>
      <td>5</td>
      <td>0.1697</td>
      <td>-0.0018</td>
      <td>0.1697</td>
      <td>-0.0142</td>
      <td>-0.0070</td>
    </tr>
    <tr>
      <th>2026-09-24</th>
      <td>3</td>
      <td>0.0135</td>
      <td>0.0667</td>
      <td>-0.0151</td>
      <td>0.0065</td>
      <td>0.0073</td>
    </tr>
    <tr>
      <th>2026-09-25</th>
      <td>18</td>
      <td>-0.1750</td>
      <td>0.0213</td>
      <td>-0.2719</td>
      <td>0.0022</td>
      <td>-0.0032</td>
    </tr>
    <tr>
      <th>2026-09-28</th>
      <td>8</td>
      <td>0.1346</td>
      <td>0.0031</td>
      <td>0.1346</td>
      <td>-0.0097</td>
      <td>-0.0022</td>
    </tr>
    <tr>
      <th>2026-09-29</th>
      <td>12</td>
      <td>-0.4252</td>
      <td>0.0833</td>
      <td>-0.3192</td>
      <td>0.0065</td>
      <td>0.0083</td>
    </tr>
    <tr>
      <th>2026-09-30</th>
      <td>7</td>
      <td>0.6675</td>
      <td>0.0054</td>
      <td>0.6675</td>
      <td>0.0001</td>
      <td>0.0022</td>
    </tr>
  </tbody>
</table>
</div>



    
![png](hackernews_notebook_export_files/hackernews_notebook_export_4_3.png)
    



<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>lag-1 F-test p-value</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>compound -&gt; excess return</th>
      <td>0.654283</td>
    </tr>
    <tr>
      <th>risk intensity -&gt; excess return</th>
      <td>0.448352</td>
    </tr>
    <tr>
      <th>post count -&gt; excess return</th>
      <td>0.767994</td>
    </tr>
    <tr>
      <th>weighted compound -&gt; excess return</th>
      <td>0.715943</td>
    </tr>
  </tbody>
</table>
</div>


    Interpretation: p-values are descriptive low-power diagnostics, not causal estimates. A zero on a no-post trading day means no observed sample activity, not neutral public sentiment.


## Results-based interpretation

### What the Hacker News sample says

The fixed-query collector returned **211 unique Hacker News stories/comments** across **23 calendar days** from September 8 through September 30 (the notebook's `END` date is exclusive). The sample is therefore sustained rather than a one-day burst, but activity is uneven: the highest daily counts were 29 posts on September 12, 25 on September 9, and 19 on September 25. The initial September 8 date has only three observations, so it should not be used to characterize a platform-wide opening-day reaction.

Across all collected observations, mean VADER compound sentiment is **-0.035** and mean risk intensity is **0.0163**. Read together, these show mildly negative, risk-tilted language on average - not uniformly catastrophic discourse. Tone changes materially from day to day: the trading-day mean is most negative on September 29 (-0.425, 12 posts), September 14 (-0.380, eight posts), and September 10 (-0.280, 18 posts), while September 30 (+0.668, seven posts), September 15 (+0.430, four posts), and September 22 (+0.827, three posts) are positive but thin. September 27 remains a weekend observation and is excluded from the trading-day return panel; September 29 and September 30 are now paired with their completed market returns.

### Market comparison

For the **16 aligned trading observations** (September 9-30), the equal-weight AI basket compounded **+2.88%** while SPY compounded **-0.19%**. The basket's positive window-level performance coexists with several negative Hacker News sentiment days. That pattern is inconsistent with a simple claim that more negative discussion mechanically depressed AI equities. It does not demonstrate the opposite: daily returns also reflect broad-market, macroeconomic, earnings, and company-specific information that this bivariate exercise does not control for.

### Granger diagnostics

None of the four one-lag tests rejects the no-predictive-content null for next-day basket excess return: unweighted compound sentiment **p = 0.654**, risk intensity **p = 0.448**, post count **p = 0.768**, and engagement-weighted compound sentiment **p = 0.716**. The most cautious reading is that this query-defined Hacker News sample supplies **no detectable incremental predictive signal** for next-day AI-basket excess returns in this short window. These are not tests of structural causality and have low power with 16 aligned days; non-rejection is not evidence that discussion has no economic relevance.

### Interpretation limits

Hacker News is a technically oriented, self-selected community, not a representative cross-section of investors or the public. The collector mixes comments and stories, and the four fixed search phrases preferentially retrieve risk- and Anthropic-related discussion. VADER and the transparent lexicon may miss sarcasm, technical nuance, and quoted language; the median observed engagement is only 1, so engagement-weighted results should be treated as a sensitivity check rather than a primary measure. The appropriate conclusion is source-specific: Hacker News discussion was episodically negative and risk-oriented, but this sample does not show a reliable lagged association with the selected AI basket's excess returns.


## Interpretation

Use source-level daily counts and sentiment only with the displayed query, date window, and collection metadata. Do not pool this sample with another platform without preserving the source label and explaining the selection rule.
