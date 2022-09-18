<script lang="ts">
	//import { JSDOM } from 'jsdom';
	import type { Item } from 'src/item';
	import { page } from '$app/stores';
	import { onMount } from 'svelte';
	import { fade } from 'svelte/transition';

	const MININTERVAL = 5;

	let searchUrl = '';
	let searchUrlInput: string;
	let blacklistInput: string;
	let error = '';
	let timer: any = undefined;
	let interval: number;
	let items: Item[] = [];
	let fetching = false;

	onMount(() => {
		searchUrlInput = $page.url.searchParams.get('url') ?? '';
		blacklistInput = $page.url.searchParams.get('block') ?? '';
		interval = parseInt($page.url.searchParams.get('interval') ?? MININTERVAL.toString());
		if (searchUrlInput) startSearch();
	});

	function startSearch() {
		if (timer) {
			clearInterval(timer);
			timer = undefined;
			return;
		}

		if (interval < MININTERVAL)
			return (error = `Interval muss mindestens ${MININTERVAL}s betragen`);

		if (searchUrlInput.match(/https:\/\/www\.ebay-kleinanzeigen\.de\/.*/)) {
			searchUrl = searchUrlInput;
			items = [];
			if (timer) clearInterval(timer);
			timer = setInterval(lookup, interval * 1000);
			lookup();
		} else {
			error = 'Not a valid ebay url!';
		}
	}

	async function lookup() {
		fetching = true;
		error = '';
		let blacklist: string[] = blacklistInput.trim().length ? blacklistInput.split(/\s/) : [];
		const $ = (window as any).$;
		let toAdd: Item[] = [];
		$.get('https://corsproxy.io/?' + searchUrl, (html: any) => {
			$(html)
				.find('.aditem')
				.each(function (this: any) {
					if ($(this).find('.icon-feature-topad').length) return;

					let item: Item = {
						url: $(this).attr('data-href'),
						price: $(this).find('.aditem-main--middle--price-shipping--price').text().trim(),
						title: $(this).find('.ellipsis').text(),
						img: $(this).find('.imagebox').attr('data-imgsrc'),
						location: $(this)
							.find('.aditem-main--top--left')
							.text()
							.replace(/(\n|\s)+/g, ' ')
							.trim(),
						time: $(this)
							.find('.aditem-main--top--right')
							.text()
							.replace(/(\n|\s)+/g, ' ')
							.trim(),
						desc: $(this).find('.aditem-main--middle--description').text(),
						tags: new Array<string>()
					};
					$(this)
						.find('.simpletag')
						.each(function (this: any) {
							item.tags.push($(this).text());
						});

					if (
						!items.some((it) => it.url == item.url) &&
						!blacklist.some((b) =>
							(item.title + item.desc + item.location).toLowerCase().includes(b)
						)
					) {
						toAdd.push(item);
					}
				});
			items = [...toAdd, ...items];
			fetching = false;
		});
	}
</script>

<div class="flex align-middle items-center flex-col">
	<form class="m-5" on:submit|preventDefault={startSearch}>
		<input
			class="
        form-control
        px-3
        py-1.5
        text-base
        font-normal
        text-gray-700
        bg-white bg-clip-padding
        border border-solid border-gray-300
        rounded
        transition
        ease-in-out
        m-0
        focus:text-gray-700 focus:bg-white focus:border-blue-600 focus:outline-none
      "
			bind:value={searchUrlInput}
			placeholder="Kleinanzeigen Such URL"
		/>
		<span class="ml-2">Interval:</span>
		<input
			class="form-control
        px-3
		ml-1
		w-16
        py-1.5
        text-base
        font-normal
        text-gray-700
        bg-white bg-clip-padding
        border border-solid border-gray-300
        rounded
        transition
        ease-in-out
        m-0
        focus:text-gray-700 focus:bg-white focus:border-blue-600 focus:outline-none"
			type="number"
			bind:value={interval}
			placeholder="Time between queries in s"
		/>
		<span>s</span>
		<input
			class="
        form-control
        px-3
        py-1.5
        text-base
        font-normal
        text-gray-700
        bg-white bg-clip-padding
        border border-solid border-gray-300
        rounded
        transition
        ease-in-out
        m-0
		ml-3
        focus:text-gray-700 focus:bg-white focus:border-blue-600 focus:outline-none
      "
			bind:value={blacklistInput}
			placeholder="Blacklist (Leerzeichen trennen)"
		/>
		<input
			type="submit"
			value={timer ? 'Pause' : 'Start'}
			class="inline-block ml-3 px-6 py-2.5 bg-blue-600 text-white font-medium text-xs leading-tight uppercase rounded shadow-md hover:bg-blue-700 hover:shadow-lg focus:bg-blue-700 focus:shadow-lg focus:outline-none focus:ring-0 active:bg-blue-800 active:shadow-lg transition duration-150 ease-in-out"
		/>
	</form>
	<div class="h-6">
		{#if fetching}
			<div class="flex items-center gap-2 text-gray-500">
				<span class="h-6 w-6 block rounded-full border-4 border-t-blue-300 animate-spin" />
				fetching...
			</div>
		{/if}
	</div>
	{#if error}
		<p>{error}</p>
	{/if}
</div>

<!-- <h1 class="text-3xl font-bold underline">Hello world!</h1> -->

<a
	href={`/ebay_list?url=${searchUrlInput}&interval=${interval}&block=${blacklistInput}`}
	class="fixed right-2 bottom-2 bg-blue-600 p-3 font-medium text-xs leading-tight uppercase rounded shadow-md hover:bg-blue-700 hover:shadow-lg"
>
	Lesezeichen
</a>

<div class="grid grid-cols-1 p-2 md:grid-cols-2 xl:grid-cols-3">
	{#each items as item}
		<a transition:fade href={'https://www.ebay-kleinanzeigen.de' + item.url} target="__blank">
			<div class="grid grid-cols-[1fr_2fr] m-3 rounded hover:bg-slate-50">
				<div class="aspect-square rounded flex items-center justify-center bg-slate-100">
					<img src={item.img} class="max-h-full max-w-full" alt="" />
				</div>
				<div class="m-3 flex flex-col place-content-around">
					<div>
						<h2 class="text-xl font-bold">{item.title}</h2>
						<p>
							<span class="text-xl"> {item.price} </span>
							{#each item.tags as tag}
								<span class="text-right">{tag}</span>
							{/each}
						</p>
					</div>
					<div>
						<p>{item.location}</p>
						<p>{item.time}</p>
					</div>
				</div>
			</div>
		</a>
	{/each}
</div>
