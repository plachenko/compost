<script lang="ts">
    import RadialProgress from "./RadialProgress.svelte";

    const {
        btnDisabled,
        loading,
        loadText,
        promptSendHandler,
    }: {
        btnDisabled: boolean;
        loading: boolean;
        loadText: string;
        promptSendHandler: () => void;
    } = $props();
</script>

<button
    disabled={btnDisabled || false}
    onclick={promptSendHandler}
    class="
    justify-center
    flex
    ">
    {#if !loading}
        {loadText}
    {:else}
        <RadialProgress meter={"#FF3E00"} />
    {/if}
</button>