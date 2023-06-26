<svelte:options tag="taocode-rss-display" />

<script>
	import { onMount, setContext } from 'svelte'
	
	export let feeds
	export let dedupe = true
	export let cache = true
	export let cachetimeout = .03*60*1000
	export let corsproxy = ''
	export let maxperfeed = 5

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
	
	function getContent(xmlObj, tagName) {
    return xmlObj.getElementsByTagName(tagName)[0].textContent
  }
	let CACHE_ARTICLES = 'rssfeedcache'
	let CACHE_LAST = 'rssfeedlastfetch'
	let articles = []
	let lastFetch = localStorage.getItem(CACHE_LAST) || 0

	onMount(async () => {
		console.log('mount!')
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
		await Promise.all(
			parsedFeeds.map(
				async feed => {
					console.log('fetching:',`${corsproxy}${feed.src}`)
					const response = await fetch(`${corsproxy}${feed.src}`)
					const text = await response.text()
					const data = (new window.DOMParser()).parseFromString(text, "text/xml")
					let channel = data.getElementsByTagName('channel')
					console.log({channel,data,text})
					let items = Array.prototype.slice.call(channel[0].children)
					console.log('items count:',items.length,{items})
					let feedArticleCount = 0
					items.forEach(item => {
						if (item.tagName === 'item') {
							console.log(feed.tag,'Item:',getContent(item,'title'),{item})
							const titleEl = document.createElement("h3")
							titleEl.innerHTML = getContent(item, 'title')
							const titleText = titleEl.innerText
							const descEl = document.createElement("div")
							descEl.innerHTML = getContent(item, 'description')
							const extant = articles.some(a => a.title === titleText)
							if (dedupe && extant) {
								articles = articles.map(c => {
									if (c.title === titleText 
											&& ! c.tags.some(a => a[0] === feed.tag))
										c.tags.push([feed.tag,feed.taglink])
									return c
								})
								return
							}
							if (feedArticleCount >= maxperfeed) return
							articles.push({
								'feed': feed.title,
								'title': titleEl.innerText,
								'description': descEl.innerText,
								'href': getContent(item, 'link'),
								'date': new Date(getContent(item, 'pubDate')),
								'tags': [[feed.tag,feed.taglink]],
								// 'thumbnail': item.getElementsByTagName('enclosure')[0].getAttribute('url'),
								// 'source': item.getElementsByTagName('source')[0].getAttribute('url')
							})
							feedArticleCount++
						}
					})
				}
			)
		)
		articles = articles.sort((a,b) => b.date - a.date)
		localStorage.setItem(CACHE_ARTICLES, JSON.stringify(articles))
		localStorage.setItem(CACHE_LAST, Date.now())
		console.log('end of mount')
  })
	$: show = articles.length
	const dtFormat = new Intl.DateTimeFormat("en", {
		dateStyle: 'medium',
		// timeStyle: 'long'
	})
</script>

<div>total: {articles.length}</div>
<ul>
	{#each articles as a}
	<li class="{a.tags.reduce((p,c) => `${p} ${c}`)}">
		<div class="date-tags">
			<span class="pubdate">{dtFormat.format(a.date)}</span>
			{#each a.tags as t}
			{#if t.length > 1 && t[1]}
				<a href="{t[1]}"><span class="tag">{t[0]}</span></a>
			{:else}
			<span class="tag">{t[0]}</span>
			{/if}
			{/each}
		</div>
		<a class="link" href="{a.href}">
			<h3>{a.title}</h3>
			<p>{a.description}</p>
		</a>
	</li>
	{/each}
</ul>

<style>
	a {
		text-decoration: none;
	}
	a:hover h3,
	a:hover span {
		text-decoration: underline;
	}
	.link {
		display: block;
	}
	h3,p {
		margin: 0.4em 0 0.3em;
	}
	ul {
		list-style-type: none;
		padding: 0;
	}
	.date-tags {
		margin-top: 2em;
	}
	.pubdate {
		font-size: 0.9em;
	}
	.tag {
		font-size: 0.7em;
		margin: 0 0.25em;
		padding: 0.2em 0.4em;
		border-radius: 0.2rem;
		background-color: #CCC7;
		text-transform: uppercase;
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
