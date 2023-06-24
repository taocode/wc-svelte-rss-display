<svelte:options tag="taocode-rss-display" />

<script>
	import { onMount, setContext } from 'svelte'
	
	export let feeds

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
</script>

<style>
	span {
		width: 4rem;
		display: inline-block;
		text-align: center;
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

<div>Here's the feeds:</div>
<ul>
	{#each parsedFeeds as f}
	<li>{f.title} - {f.src}</li>
	{/each}
</ul>