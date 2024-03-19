<script lang="ts">
    /* === IMPORTS ============================ */
    import type { Mode, Color } from "../routes/canvas/+page.svelte";

    /* === PROPS ============================== */
    export let mode: Mode; // bind
    export let color: Color; //bind

    /* === CONSTANTS ========================== */
    const colors: {name: string, colorRGB: Color}[] = [
        {name: "Black", colorRGB: [26, 26, 26]},
        {name: "White", colorRGB: [240, 240, 240]},
        {name: "Red", colorRGB: [245, 73, 61]},
        {name: "Green", colorRGB: [140, 217, 63]},
        {name: "Blue", colorRGB: [59, 175, 237]},
        {name: "Yellow", colorRGB: [250, 229, 40]}
    ];
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

    {#if mode === "draw"}
        <div role="group" aria-label="Color:" id="color">
            {#each colors as {name, colorRGB}}
                <div 
                    class="colorInput"
                    style="--_clr: rgb({colorRGB[0]}, {colorRGB[1]}, {colorRGB[2]})">
                    <input
                        type="radio"
                        id={name}
                        class="visuallyHidden"
                        name="color"
                        value={colorRGB}
                        bind:group={color} />
                    <label for={name}>
                        <span class="visuallyHidden">{name}</span>
                    </label>
                </div>
            {/each}
        </div>
    {/if}
</form>



<style lang="scss">
    $_clr-600: #6e6e6e;
    $_clr-bg: #ffffff;
    $_colorInput-size: 44px;

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

        #color {
            display: grid;
            grid-template-columns: repeat(auto-fit, minmax($_colorInput-size, 1fr));

            border: solid 1px $_clr-600;
            border-top: none;

            input:checked + label {
                &::before {
                    content: '';
                    position: absolute;
                    top: 12px;
                    right: 12px;
                    bottom: 12px;
                    left: 12px;
                    
                    background-color: #ffffff;
                    border: solid 1px $_clr-600;
                }
            }

            label {
                display: block;
                position: relative;
                height: $_colorInput-size;

                background-color: var(--_clr);
                cursor: pointer;
            }
        }
    }
</style>