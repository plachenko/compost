<script lang="ts">
    import { onMount, onDestroy } from "svelte";

    let { handleSpeech, setListen } = $props();

    let recognition;
    let isListening = $state(false);
    let transcript = "";

    onMount(() => {
        if (
            !("webkitSpeechRecognition" in window) &&
            !("SpeechRecognition" in window)
        ) {
            alert("Your browser does not support Speech Recognition API");
            return;
        }

        const SpeechRecognition =
            window.SpeechRecognition || window.webkitSpeechRecognition;
        recognition = new SpeechRecognition();
        recognition.continuous = false;
        recognition.interimResults = false;
        recognition.lang = "en-US";

        recognition.onresult = (event) => {
            transcript =
                event.results[event.results.length - 1][0].transcript.trim();
        };

        recognition.onstart = () => {};

        recognition.onend = () => {
            handleSpeech(transcript);
            isListening = false;
        };
    });

    onDestroy(() => {
        if (recognition) {
            recognition.stop();
            recognition = null;
        }
    });

    export function toggleListening() {
        if (isListening) {
            recognition.stop();
        } else {
            recognition.start();
        }
        isListening = !isListening;
    }
</script>

<button
    class="bg-blue-500 text-white w-[95px] py-2 rounded relative"
    onclick={toggleListening}>
    {#if isListening}
        <div class="rounded-full absolute top-1 right-1 bg-red-600 size-[8px]">
        </div>
        <div
            class="rounded-full absolute top-1 right-1 bg-red-400 size-[8px] animate-ping">
        </div>
        Stop
    {:else}
        Listen
    {/if}
</button>

<style>
</style>