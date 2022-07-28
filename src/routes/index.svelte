<script lang="ts">
	//import { JSDOM } from 'jsdom';
	import type { Item } from 'src/item';

	let searchUrl = '';
	let searchUrlInput = '';
	let error = '';
	let timer: any = undefined;
	let interval = 15;
	let items: Item[] = [];
	let fetching = false;

	function startSearch() {
		if (timer) {
			clearInterval(timer);
			timer = undefined;
			return;
		}

		if (searchUrlInput.match(/https:\/\/www\.ebay-kleinanzeigen\.de\/.*/)) {
			searchUrl = searchUrlInput;
			if (timer) clearInterval(timer);
			timer = setInterval(lookup, interval * 1000);
			lookup();
		} else {
			error = 'Not a valid ebay url!';
		}
	}

	async function lookup() {
		//let res = await fetch(searchUrl).then((res) => res.text());

		//const dom = new JSDOM(res);
		fetching = true;
		error = '';
		const $ = (window as any).$;
		$.get('https://api.allorigins.win/raw?url=' + searchUrl, (html: any) => {
			$(html)
				.find('.aditem')
				.each(function (this: any) {
					let item: Item = {
						url: $(this).attr('data-href'),
						price: $(this).find('.aditem-main--middle--price').text().trim(),
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
					// if (
					// 	!config.blackList.some((b) =>
					// 		(item.title + item.desc + item.location).toLowerCase().includes(b)
					// 	)
					// )
					if (!items.some((it) => it.url == item.url)) {
						items = [item, ...items];
					}
				});
			//console.log(items);
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
			placeholder="Ebay Such URL"
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

<div class="grid grid-cols-3 p-2">
	{#each items as item}
		<a href={'https://www.ebay-kleinanzeigen.de' + item.url} target="__blank">
			<div class="grid grid-cols-2 m-3">
				<div class="m-3">
					<h2 class="text-xl font-bold">{item.title}</h2>
					<p>
						<span class="text-xl"> {item.price} </span>
						{#each item.tags as tag}
							<span class="text-right">{tag}</span>
						{/each}
					</p>
					<p>{item.location}</p>
					<p>{item.time}</p>
				</div>
				<img src={item.img} alt="" />
			</div>
		</a>
	{/each}
</div>
