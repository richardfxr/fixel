<script lang="ts">
    /* === IMPORTS ============================ */
    import type { Mode } from "../routes/canvas/+page.svelte";

    /* === PROPS ============================== */
    export let mode: Mode; // bind
</script>



<form>
    <div role="group" aria-label="Mode:" id="mode">
        <div>
            <input
                type="radio"
                id="draw"
                class="visuallyHidden"
                name="mode"
                value="draw"
                bind:group={mode} />
            <label for="draw">Draw</label>
        </div>
        <div>
            <input
                type="radio"
                id="move"
                class="visuallyHidden"
                name="mode"
                value="move"
                bind:group={mode} />
            <label for="move">Move</label>
        </div>
    </div>
</form>



<style lang="scss">
    $_clr-600: #6e6e6e;
    $_clr-bg: #ffffff;

    form {
        display: flex;
        flex-flow: column nowrap;
        align-items: center;
        position: fixed;
        top: 10px;
        right: 0;
        left: 0;
        z-index: 2;
        
        padding: 0 10px;
        pointer-events: none;

        & > div {
            display: flex;
            flex-flow: row wrap;
            width: 100%;
            max-width: 300px;
            background-color: $_clr-bg;

            pointer-events: auto;
        }

        #mode {
            display: grid;
            grid-template-columns: 1fr 1fr;

            border: solid 1px $_clr-600;

            input:checked + label {
                color: white;
                background-color: $_clr-600;
            }

            label {
                display: block;
                width: 100%;
                padding: 6px 15px;
                text-align: center;

                color: $_clr-600;
                cursor: pointer;
            }
        }
    }
</style>