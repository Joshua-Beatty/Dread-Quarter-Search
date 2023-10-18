<script lang="ts">
    import TrieSearch from "trie-search";
    import sleep from "sleep-promise";
    import { onMount } from "svelte";

    enum State {
        Loading,
        Loaded,
        Displaying,
        Clear,
    }
    let state = State.Loading;

    type Card = {
        name: string;
    };

    let trie: TrieSearch<Card>;
    async function loadData() {
        console.log("Fetching Data");
        const response = await fetch(
            "https://www.dreadquarter.com/season-1-cards.txt"
        );
        const cards = await response.text();
        console.log("Processing Data");
        trie = new TrieSearch<Card>("name");
        trie.addAll(
            cards.split("\n").map((x) => {
                return { name: x.trim() };
            })
        );
        state = State.Loaded;
        console.log("Loaded!");
    }
    onMount(loadData);

    let searchText = "";
    let sort = "name";
    let direction = "auto";
    let searching = false;
    let results: any[] = [];
    let lastSearchTimeout: Promise<any>;
    let nextPage = "";
    let total = "";
    let hasMore = false;
    async function search(e?: any, q?: string) {
        searching = true;
        results = [];
        total = "";
        if (lastSearchTimeout) await lastSearchTimeout;
        lastSearchTimeout = sleep(500);
        try {
            const searchQuery = encodeURIComponent(
                `(legal:pioneer or banned:pioneer) and (${searchText})`
            );
            const newUrl =
                q ||
                `https://api.scryfall.com/cards/search?order=${sort}&dir=${direction}&q=${searchQuery}`;
            const response = await fetch(newUrl);
            const json = await response.json();
            nextPage = json.has_more ? json.next_page : "";
            total = json.total_cards;
            hasMore = json.has_more;
            results = json.data.filter((x: any) => {
                return trie.search(x.name)?.[0]?.name == x.name;
            });
            console.log(results);
            results = results;
        } catch (e) {
            throw e;
        } finally {
            searching = false;
        }
        console.log(nextPage);
    }

    async function goNextPage() {
        const temp = nextPage;
        nextPage = "";
        await search({}, temp);
    }
    let resultType = "images";
</script>

<svelte:head>
    <title>DreadQuarter Scryfall Search</title>
</svelte:head>
{#if state === State.Loading}
    <div
        class="loadingContainer"
        style="display: flex;
align-items: center;
justify-content: center;
height: 100vh;"
    >
        <p style="font-size: 40px;">Loading Data...</p>
    </div>
{:else if state === State.Loaded}
    <div style="text-align: center;">
        <h1>DreadQuarter Scryfall Search</h1>
    </div>
    <form class="pure-form mainForm">
        <input bind:value={searchText} style="width: min(50em, 90vw);" /><a
            style="searchQuestionMark"
            href="https://scryfall.com/docs/syntax"
            target="_blank"
            ><span class="searchQuestionMark" title="Scryfall Search Syntax"
                >?</span
            ></a
        ><br /><br />

        <select bind:value={sort}>
            <option value="name" selected>Name</option>
            <option value="released">Release Date</option>
            <option value="set">Set/Number</option>
            <option value="rarity">Rarity</option>
            <option value="color">Color</option>
            <option value="usd">Price: USD</option>
            <option value="tix">Price: Tix</option>
            <option value="eur">Price: EUR</option>
            <option value="cmc">Mana Value</option>
            <option value="power">Power</option>
            <option value="toughness">Toughness</option>
            <option value="edhrec">EDHREC Rank</option>
            <option value="penny">Penny Dreadful Rank</option>
            <option value="artist">Artist Name</option>
            <option value="review">Set Review</option>
        </select>

        <select bind:value={direction}>
            <option value="auto" selected>Auto</option>
            <option value="asc">Ascending</option>
            <option value="desc">Descending</option>
        </select>

        <select bind:value={resultType}>
            <option value="images" selected>Images</option>
            <option value="text">Text</option>
        </select>
        <button
            disabled={searching || null}
            on:click={search}
            class="pure-button pure-button-primary">Search</button
        >
    </form>
    <div style="text-align: center;">
        {`${
            total
                ? `Displaying: ${results.length} Has More: ${
                      hasMore ? "Yes" : "No"
                  }`
                : ""
        }`}
    </div>
    {#if resultType == "images"}
        <div class="cardHolderHolder">
            <div class="cardHolder">
                {#each results as result}
                    <div class="card">
                        <a href={result.scryfall_uri} target="_blank">
                            <img
                                src={result?.image_uris?.normal ||
                                    result?.card_faces?.[0]?.image_uris?.normal}
                                alt={`${result.name} card image`}
                            /></a
                        >
                    </div>
                {/each}
            </div>
        </div>
    {:else}
        <div class="textHolder">
            {#each results as result}
                <div class="cardName">
                    {result.name}
                </div>
            {/each}
        </div>
    {/if}

    {#if nextPage}
        <form class="pure-form secondForm">
            <button
                on:click={goNextPage}
                class="pure-button pure-button-primary">Next Page</button
            >
        </form>
    {/if}
{/if}

<style>
    .secondForm {
        text-align: center;
        padding: 1em;
    }
    .mainForm {
        text-align: center;
        padding: 1em;
    }
    .card {
        width: calc((min(1200px, 100vw) - 10px * 4) / 4);
        margin: 5px;
    }
    .card img {
        max-width: 100%;
        max-height: 100%;
    }
    .textHolder {
        padding-left: 3em;
    }
    .cardHolder {
        display: flex;
        flex-wrap: wrap;
        width: min(1200px, 100vw);
        margin: 0 auto;
    }
    .searchQuestionMark {
        position: relative;
        top: -16px;
        color: blue;
    }
    a:link {
        text-decoration: none;
    }

    a:visited {
        text-decoration: none;
    }

    a:hover {
        text-decoration: none;
    }

    a:active {
        text-decoration: none;
    }
</style>
