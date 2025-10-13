<script>
    let imageUrl = "";
    import { vision } from "$lib/OpenAI";

    let fileNames = $state([]);
    let fileLoaded = $state(false);
    let fileInp = $state(null);
    let fileArr = $state([]);

    let { setImages } = $props();

    function unloadFile() {
        fileNames = [];
        fileInp = "";
        fileLoaded = false;
        fileArr = [];
    }

    function handleFileChange(event) {
        const files = event.target.files;
        const fileReadPromises = [];

        if (files.length) {
            for (let fileInt = 0; fileInt <= files.length - 1; fileInt++) {
                const file = files[fileInt];
                const filePromise = new Promise((res, rej) => {
                    fileNames.push(file.name);

                    const fileReader = new FileReader();

                    fileReader.onload = function (e) {
                        const imageBlob = e.target.result;
                        // fileArr.push(imageBlob);
                        //res(imageBlob);
                        // console.log(fileArr);

                        setImages(imageBlob);
                    };

                    fileReader.onerror = function (e) {
                        console.log("error reading file", e);
                        rej(file.name);
                    };

                    fileReader.readAsDataURL(file);
                });

                fileReadPromises.push(filePromise);

                /*
                Promise.all(filePromise)
                    .then((fname) => {
                        console.log("processed.", fname);
                        vision("What's in this image", fileArr);
                    })
                    .catch((err) => {
                        console.log("Error:", err);
                    });
                */
            }

            // imageUrl = URL.createObjectURL(file);
        }
    }
</script>

<div class="p-1 flex gap-1">
    <div class="flex-1 flex flex-row gap-1">
        <button class="flex items-center p-4">📷</button>
        <input
            bind:value={fileInp}
            type="file"
            id="file-upload"
            class="hidden"
            accept="image/*"
            multiple
            onchange={handleFileChange} />

        {#if !fileNames.length}
            <label
                for="file-upload"
                class="flex-1 cursor-pointer bg-slate-300 flex items-center justify-center rounded-md p-2"
                >Select Image(s)</label>
        {:else}
            <button onclick={() => unloadFile()} class="flex items-center"
                >Clear</button>
            <label
                for="file-upload"
                class="flex-1 cursor-pointer bg-slate-300 flex items-center justify-center rounded-md p-2"
                >Add</label>
        {/if}

        {#if fileNames.length}
            <span
                class="flex-1 cursor-pointer bg-slate-300 flex items-center justify-center rounded-md p-2"
                >{fileNames[0]}
            </span>
            {#if fileNames.length > 1}
                <button class="flex items-center justiy-center"
                    >+ {fileNames.length - 1}</button>
            {/if}
        {/if}
    </div>

    {#if fileLoaded}
        <button
            class="flex items-center justify-Center"
            onclick={() => {
                unloadFile();
            }}>X</button>
    {/if}
</div>

<style>
    .preview {
        max-width: 100%;
        height: auto;
        margin-top: 10px;
    }
</style>