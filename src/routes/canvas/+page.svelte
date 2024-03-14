<script lang="ts">
    /* === IMPORTS ============================ */
    import Canvas from "$lib/canvas.svelte";

    /* === CONSTANTS ========================== */
    const width = 6;
    const height = 6;
    let image = generateGrid(width, height);

    /* === FUNCTIONS ========================== */
    function generateGrid(width: number, height: number): Uint8ClampedArray {
        const grid = new Uint8ClampedArray(width * height * 4);
        for (let i = 0; i < width * height; i++) {
            grid[i * 4] = (i % width + Math.floor(i / width)) % 2 * 115 + 110;
            grid[i * 4 + 1] = (i % width + Math.floor(i / width)) % 2 * 115 + 110;
            grid[i * 4 + 2] = (i % width + Math.floor(i / width)) % 2 * 115 + 110;
            grid[i * 4 + 3] = 255;
        }
        return grid;
    }
</script>



<main>
    <Canvas
        {width}
        {height}
        bind:image={image} />
</main>



<style lang="scss">
    main {
        background-color: white;
    }
</style>