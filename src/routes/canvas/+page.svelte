<script lang="ts" context="module">
    /* === TYPES ============================== */
    export type Mode = "draw" | "move";
    export type Color = [number, number, number];
</script>

<script lang="ts">
    /* === IMPORTS ============================ */
    import Canvas from "$lib/canvas.svelte";
    import CanvasControls from "$lib/canvasControls.svelte";

    /* === CONSTANTS ========================== */
    const width = 9;
    const height = 9;

    /* === VARIABLES ========================== */
    let image = generateGrid(width, height);
    let mode: Mode = "draw";
    let color: Color = [0, 0, 0];

    /* === FUNCTIONS ========================== */
    function generateGrid(width: number, height: number): Uint8ClampedArray {
        const grid = new Uint8ClampedArray(width * height * 4);
        for (let i = 0; i < width * height; i++) {
            grid[i * 4] = (i % width + Math.floor(i / width)) % 2 * 80 + 150;
            grid[i * 4 + 1] = (i % width + Math.floor(i / width)) % 2 * 80 + 150;
            grid[i * 4 + 2] = (i % width + Math.floor(i / width)) % 2 * 80 + 150;
            grid[i * 4 + 3] = 255;
        }
        return grid;
    }
</script>



<main>
    <CanvasControls
        bind:mode={mode}
        bind:color={color} />
    
    <Canvas
        {width}
        {height}
        bind:image={image}
        {mode}
        {color} />
</main>



<style lang="scss">
    main {
        color: black;
        background-color: white;
    }
</style>