<svelte:options tag="taocode-rss-display" />

<script>
	import { onMount, setContext } from 'svelte'
	
	export let feeds
	export let dedupe = true
	export let cache = true
	export let cachetimeout = 3*60*1000

	let parsedFeeds = []
	try {
		parsedFeeds = JSON.parse(feeds)
	} catch (err) {
		console.warn(`Feeds isn't JSON, will try single URL: ${feeds}`,err.message)
	}
	if (!parsedFeeds.length) {
		try {
			const checkURL = new URL(feeds) // throws error if malformed
			if (checkURL.host && /^https?:/.test(checkURL.protocol)) parsedFeeds.push(feeds)
			else console.warn(`Feed URL: "${feeds}" is malformed, it must be a full URL that starts with https:// or http:// and is otherwise properly shaped`)
		} catch (err) {
			console.warn(`Feed: "${feeds}" is malformed, see docs`,err.message)
		}
	}
	let show = true
	const cors = 'https://proxy-j4bychtceq-uc.a.run.app/'
	function getContent(xmlObj, tagName) {
    return xmlObj.getElementsByTagName(tagName)[0].textContent
  }
	let CACHE_ARTICLES = 'rssfeedcache'
	let CACHE_LAST = 'rssfeedlastfetch'
	let articles = []
	let lastFetch = localStorage.getItem(CACHE_LAST) || 0

	onMount(async () => {
		const timeDelta = Date.now() - lastFetch
		if (cache && timeDelta < cachetimeout * 2) {
			articles = JSON.parse(localStorage.getItem(CACHE_ARTICLES))
			console.log(`${articles.length} articles from ${parseInt(timeDelta/(60*1000))}m old cache`)
			articles = articles.map(a => {
				a.date = new Date(a.date)
				return a
			})
			if (timeDelta < cachetimeout) {
				console.log('cached articles are fresh')
				return
			} else {
				console.log('fetching fresh articles')
			}
		}
		await Promise.all(parsedFeeds.map(
			async f => {
				// console.log('fetching:',f.src)
				const response = await fetch(`${cors}${f.src}`)
				const text = await response.text()
				const data = (new window.DOMParser()).parseFromString(text, "text/xml")
				let channel = data.getElementsByTagName('channel')
				// console.log({channel,data})
				let items = Array.prototype.slice.call(channel[0].children)
				items.forEach(item => {
					if (item.tagName === 'item') {
						// console.log('Item:',getContent(item,'title'),{item})
						const titleEl = document.createElement("h3")
						titleEl.innerHTML = getContent(item, 'title')
						const titleText = titleEl.innerText
						const descEl = document.createElement("div")
						descEl.innerHTML = getContent(item, 'description')
						const extant = articles.some(a => a.title === titleText)
						if (dedupe && extant) {
							articles = articles.map(c => {
								if (c.title === titleText && ! c.tag.includes(f.tag)) c.tag += ' ' + f.tag
								return c
							})
							return
						}
						articles.push({
							'feed': f.title,
							'title': titleEl.innerText,
							'description': descEl.innerText,
							'href': getContent(item, 'link'),
							'date': new Date(getContent(item, 'pubDate')),
							'tag': f.tag,
							// 'thumbnail': item.getElementsByTagName('enclosure')[0].getAttribute('url'),
							// 'source': item.getElementsByTagName('source')[0].getAttribute('url')
						})
					}
				})
			})
		)
		articles = articles.sort((a,b) => b.date - a.date)
		localStorage.setItem(CACHE_ARTICLES, JSON.stringify(articles))
		localStorage.setItem(CACHE_LAST, Date.now())
  })
	$: show = articles.length
	const dtFormat = new Intl.DateTimeFormat("en", {
		dateStyle: 'medium',
		// timeStyle: 'long'
	})
</script>

<ul>
	{#each articles as a}
	<li class="{a.tag}">
		<a href="{a.href}">
			<h3>{a.title}</h3>
			<div>
				<span class="pubdate">{dtFormat.format(a.date)}</span>
				<span class="feeds">{a.tag}</span>
			</div>
			<div>{a.description}</div>
		</a>
	</li>
	{/each}
</ul>

<style>
	a {
		display: block;
		text-decoration: none;
		padding: 0.5em;
	}
	a:hover h3 {
		text-decoration: underline;
	}
	ul {
		list-style-type: none;
		padding: 0;
	}
	.feeds,
	.pubdate {
		font-size: 0.8em;
	}

	button {
		width: 4rem;
		height: 4rem;
		border: none;
		border-radius: 10px;
		background-color: seagreen;
		color: white;
	}
</style>
