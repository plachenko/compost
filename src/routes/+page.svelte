<script lang="ts">
    
    import { setKey, chat, vision } from "$lib/OpenAI";

    import KeyInput from "$lib/components/KeyInput.svelte";
    import PromptInput from "$lib/components/PromptInput.svelte";

    import Console from "$lib/components/Claude/Console.svelte";
	import Pane from "$lib/components/Claude/Pane.svelte";

    import { onMount } from "svelte";

    let key = $state("");
    let keyValid = $state(false);
    let pageLoaded = $state(false);
    let promptDiv = $state(null);
    let promptTxt = $state("");
    let resp = $state("");
    let imgBlobArr = $state([]);
    let generating = $state(false);

    let historyArray = $state([]);
    let historyNum = $state(null);
    let historyEl = $state(null);

    onMount(async () => {
        key = window.localStorage.getItem("key") || "";
        if (key !== "") {
            keyValid = true;
        }
        setKey(key);
        pageLoaded = true;
    });

    function handleSpeech(textResponse: string) {
        promptTxt = textResponse;
        sendHandler(promptTxt);
    }


    function connectHandler(keyVal: string): void {
        keyValid = true;
        window.localStorage.setItem("key", keyVal);
    }
    
    function saveHistory() {
        window.localStorage.setItem(
            "historyArray",
            JSON.stringify(historyArray),
        );
    }

    async function sendHandler(prompt) {
        generating = true;

        if (imgBlobArr.length) {
            vision(`${prompt}`, imgBlobArr)
                .then((e) => {
                    // console.log(e);
                    console.log(e.message.content);
                    resp = e.message.content;

                    // resp = e?.content.message.content || "";
                    // historyArray.push({ prompt, resp });
                    // historyNum = historyArray.length - 1;
                    // saveHistory();
                    prompt = "";
                    imgBlobArr = [];
                })
                .catch((err) => {
                    console.log(err);
                })
                .finally(() => {
                    generating = false;
                });
        } else {
            chat(`${prompt}`)
                .then((e) => {
                    resp = e?.content || "";
                    // respCont.scrollTo(0, 0);
                    promptTxt = prompt;
                    historyArray.push({ promptTxt: prompt, resp });
                    historyNum = historyArray.length - 1;
                    saveHistory();
                    prompt = "";
                })
                .catch((err) => {
                    console.log(err);
                })
                .finally(() => {
                    generating = false;
                });
        }
    }
</script>

{#if pageLoaded}
    <div class="h-full flex flex-col w-full bg-slate-100 ">
        {#if !keyValid}
            <KeyInput {key} {connectHandler} />
        {:else}
            <Pane />
            <!--
        
            <div class="flex-1">
                <PromptInput
                    bind:this={promptDiv}
                    {handleSpeech}
                    {promptTxt}
                    {sendHandler}
                    {imgBlobArr}
                    {generating} />
            </div>
            -->
            {#if resp}
                <div class="h-1/2 flex flex-col w-full">
                    <div class="bg-slate-400 p-2 flex h-full flex-col">
                        {#if resp}
                            <div class="flex h-full flex-col overflow-y-auto">
                                <div
                                    class="flex items-center flex-1 border-l-2 border-slate-200 pl-2">
                                    <div
                                        contenteditable
                                        class="focus:bg-slate-300 focus:text-slate-800 not-focus:italic text-slate-300 bg-slate-300/20 hover:bg-slate-300 hover:text-slate-500 rounded-md h-full w-full p-2 flex items-center">
                                        {resp}
                                    </div>
                                </div>
                            </div>
                        {/if}
                    </div>
                </div>
            {/if}
        <!--
            <div class="h-full flex flex-col w-full">
                <div class="bg-slate-400 p-2 flex">
                    {#if resp}
                        <div class="flex">
                            <div
                                class="flex items-center flex-1 border-l-2 border-slate-200 pl-2">
                                <div
                                    contenteditable
                                    class="focus:bg-slate-300 focus:text-slate-800 not-focus:italic text-slate-300 bg-slate-300/20 hover:bg-slate-300 hover:text-slate-500 rounded-md h-full w-full p-2 flex items-center">
                                    {promptTxt}
                                </div>
                            </div>
                        </div>
                    {/if}
                    <div class="flex-1">
                        <PromptInput
                            bind:this={promptDiv}
                            {handleSpeech}
                            {promptTxt}
                            {sendHandler}
                            {imgBlobArr}
                            {generating} />
                    </div>
                </div>
            </div>
            -->

        {/if}
    </div>
{/if}
