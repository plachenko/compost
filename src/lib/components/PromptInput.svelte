<script lang="ts">
    import { onMount } from "svelte";
    import { fade, fly } from "svelte/transition";

    import ImageUpload from "./ImageUpload.svelte";
    import SpeechRecognition from "./SpeechRecognition.svelte";
    import LoadButton from "./LoadButton.svelte";

    let promptEl = $state(null);
    let _prompt = $state("");
    let speechComp = $state(null);
    let { imgBlobArr, sendHandler, generating, handleSpeech, promptTxt } =
        $props();
    let showImgInp = $state(false);

    $effect(() => {
        _prompt = promptTxt;
    });

    onMount(() => {
        // focusInp();
    });

    function setImages(e) {
        imgBlobArr.push(e);
    }
    export function focusInp() {
        if (promptEl) {
            promptEl.focus();
        }
    }

    function promptSendHandler() {
        sendHandler(_prompt);
    }
</script>

<div class="gap-1 flex flex-col w-full">
    <div
        class={`p-1 ${showImgInp ? "border-b-2 border-slate-300 border-dashed pb-2" : "border-none"}  gap-1 flex w-full h-full bg-slate-200`}>
        {#if !promptTxt}
            <button
                onclick={() => {
                    showImgInp = !showImgInp;
                }}>🖼️</button>
        {/if}
        <div
            class={`flex-1 flex bg-white rounded-md h-auto gap-1`}>
            <div
                class="hover:cursor-text items-center h-full flex-1 overflow-x-auto relative flex">
                
                <div
                    onkeydown={(e) => {
                        switch (e.key) {
                            case "Enter":
                                e.preventDefault();
                                promptSendHandler();
                                break;
                        }
                    }}
                    role="textbox"
                    tabindex="0"
                    bind:innerHTML={_prompt}
                    bind:this={promptEl}
                    class="p-2 w-full outline-none absolute whitespace-nowrap z-[9999] h-full flex items-center"
                    contenteditable>
                </div>
                
            </div>
            <div class="flex items-center relative w-[45px]">
                {#if _prompt}
                    <button
                        transition:fade
                        onclick={() => {
                            _prompt = "";
                            focusInp();
                        }}
                        class="hover:bg-slate-400 bg-slate-400/40 text-sm absolute l-2"
                        >X</button>
                {/if}
            </div>
        </div>
        <div class="flex flex-row gap-1 items-center">
            <SpeechRecognition bind:this={speechComp} {handleSpeech} />
            {#if _prompt !== ""}
                <LoadButton
                    btnDisabled={_prompt == "" || generating}
                    loadText={"go"}
                    loading={generating}
                    {promptSendHandler} />
            {/if}
        </div>
    </div>
    {#if showImgInp}
        <div transition:fly={{ y: -10, duration: 300 }} class="w-full">
            <ImageUpload {setImages} />
        </div>
    {/if}
</div>