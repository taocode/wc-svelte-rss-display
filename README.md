```js script
import './src/index.svelte'
```

# RSS Feed Web Component

## Articles from Feeds

```html preview-story
  <taocode-rss-display
  corsproxy='https://proxy-j4bychtceq-uc.a.run.app/'
  maxperfeed=5
  feeds='[
  {"src":"https://www.salem.edu/news/rss","title":"College","tag":"college","taglink":"https://www.salem.edu/news"},
  {"src":"https://www.salemacademy.com/rss/news","title":"Academy","tag":"academy"}
  ]'></taocode-rss-display>
```
