<script lang="ts">
    /* === IMPORTS ============================ */
    import { onMount } from 'svelte';
    import type { Mode } from '../routes/canvas/+page.svelte';

    /* === TYPES ============================== */
    type Pointer = {
        id: number,
        x: number,
        y: number
    };

    /* === PROPS ============================== */
    export let width: number;
    export let height: number;
    export let image: Uint8ClampedArray; // bind
    export let mode: Mode;

    /* === CONSTANTS ========================== */
    const panXSensitivity = 1;
    const panYSensitivity = 1;
    const pinchZoomSensitivity = 0.23;
    const mouseZoomCoefficient = -1;
    const mouseZoomLimit = 7;
    const scrollXCoefficient = -0.9;
    const scrollYCoefficient = -0.9;
    const zoomWithCtrl = true;
    const ongoingPointers: Pointer[] = [];
    const scaleMin = 10;
    const scaleMax = 100;

    /* === VARIABLES ========================== */
    let containerWidth = 100;
    let containerHeight = 100;
    let scale = 50;
    let translateX = 0;
    let translateY = 0;
    let draggingPointer: Pointer | null = null;
    let previousPinchDistance = -1;

    /* === BINDINGS =========================== */
    let container: HTMLDivElement;
    let canvas: HTMLCanvasElement;
    let context: CanvasRenderingContext2D;
    let imageData: ImageData;

    /* === FUNCTIONS ========================== */
    function clamp(input: number, min: number, max: number): number {
        return Math.min(Math.max(input, min), max);
    }

    function copyPointerEvent(pointer: PointerEvent): Pointer {
        return {
            id: pointer.pointerId,
            x: pointer.clientX,
            y: pointer.clientY,
        };
    }

    function getPointerIndex(pointer: PointerEvent): number {
        return ongoingPointers.findIndex(element => element.id === pointer.pointerId);
    }

    function translate(translateXDiff: number, translateYDiff: number): void {
        translateX = clamp(
            translateX + translateXDiff,
            -1 * (containerWidth / 2 - 1),
            containerWidth / 2 - 1
        );
        translateY = clamp(
            translateY + translateYDiff,
            -1 * (containerHeight / 2 - 1),
            containerHeight / 2 - 1
        );
    }

    function scaleOnPoint(scaleDiff: number, pointX: number, pointY: number): void {
        const newScale = clamp(scale + scaleDiff, scaleMin, scaleMax);

        // adds addtional translates so that the scale transform appear to be centered
        // on point (pointX, pointY)
        translate(
            -1 * ((pointX - containerWidth / 2) - translateX) / scale * (newScale - scale),
            -1 * ((pointY - containerHeight / 2) - translateY) / scale * (newScale - scale)
        );

        scale = newScale
    }

    function drag(event: PointerEvent): void {
        if (
            draggingPointer &&
            event.pointerId === draggingPointer.id
        ) {
            // the pointer is dragging, update translates (position)
            translate(
                panXSensitivity * (event.clientX - draggingPointer.x),
                panYSensitivity * (event.clientY - draggingPointer.y)
            );
            draggingPointer = copyPointerEvent(event);
        }
    }

    function renderNewImage(): void {
        imageData.data.set(image);
        context.putImageData(imageData, 0, 0);
    }

    /* === EVENT HANDLES ====================== */
    function handleResize(): void {
        // update width and height
        containerWidth = container.clientWidth;
        containerHeight = container.clientHeight;

        // apply transition to ensure image stays in view
        translate(0, 0);
    }

    function handleEnter(event: PointerEvent): void {
        if (event.buttons !== 0) return;
        // cancel pointer as it could still be active
        // if pointerUp or pointerCancel was never called
        handleCancel(event);
    }

    function handleDown(event: PointerEvent): void {
        // add poiner to ongoingPointers and set draggingPointer
        ongoingPointers.push(copyPointerEvent(event));
        draggingPointer = copyPointerEvent(event);
    }

    function handleMove(event: PointerEvent): void {
        const pointerIndex = getPointerIndex(event);
        if (pointerIndex < 0) return;

        // update pointer object in ongoingPointers
        ongoingPointers.splice(pointerIndex, 1, copyPointerEvent(event));

        switch (mode) {
            case "draw":
                handleDrawMode(event);
                break;
            case "move":
                handleMoveMode(event);
                break;
        }
    }

    function handleDrawMode(event: PointerEvent): void {
        // get the (x, y) pixel position of the pointer relative to the image
        const imageX = Math.floor((((event.clientX - translateX) - (containerWidth / 2)) / scale) + width / 2);
        const imageY = Math.floor((((event.clientY - translateY) - (containerHeight / 2)) / scale) + height / 2);

        // return if pointer is outside the image
        if (
            imageX < 0 || imageX >= width ||
            imageY < 0 || imageY >= height
        ) return;

        // update image
        const imageIndex = imageX + width * imageY;
        image[imageIndex * 4] = 255;
        image[imageIndex * 4 + 1] = 0;
        image[imageIndex * 4 + 2] = 0;
        image[imageIndex * 4 + 3] = 255;

        renderNewImage();
    }

    function handleMoveMode(event: PointerEvent): void {
        if (ongoingPointers.length === 2) {
            // two pointers are present, starting pinch zoom gesture
            // calculate the distance between them and the center point
            const distanceX = ongoingPointers[0].x - ongoingPointers[1].x;
            const distanceY = ongoingPointers[0].y - ongoingPointers[1].y;
            const centerX = ongoingPointers[0].x - distanceX / 2;
            const centerY = ongoingPointers[0].y - distanceY / 2;

            if (draggingPointer && draggingPointer.id === -1) {
                // the point between the two pointers is dragging
                // update translates (position)
                translate(
                    panXSensitivity * (centerX - draggingPointer.x),
                    panYSensitivity * (centerY - draggingPointer.y)
                );
            }

            // update/set dragging pointer to the point between the two pointers
            draggingPointer = {
                id: -1,
                x: centerX,
                y: centerY
            };

            const currentPinchDistance = Math.sqrt(Math.pow(distanceX, 2) + Math.pow(distanceY, 2));

            if (previousPinchDistance >= 0) {
                scaleOnPoint(pinchZoomSensitivity * (currentPinchDistance - previousPinchDistance), centerX, centerY);
            }

            previousPinchDistance = currentPinchDistance;
            return;
        }

        drag(event);
    }

    function handleUp(event: PointerEvent): void {
        switch (mode) {
            case "draw":
                handleDrawMode(event);
                break;
            case "move":
                drag(event);
                break;
        }

        handleCancel(event);
    }

    function handleCancel(event: PointerEvent): void {
        const pointerIndex = getPointerIndex(event);
        if (pointerIndex < 0) return;

        previousPinchDistance = -1;
        ongoingPointers.splice(pointerIndex, 1);

        if (
            draggingPointer &&
            draggingPointer.id === -1 &&
            ongoingPointers.length >= 1
        ) {
            // dragging was assigned to the point between two pointers (pinch zoom)
            // now defaults back to the first pointer in ongoingPointers
            draggingPointer = {...ongoingPointers[0]};
        }

        if (
            draggingPointer &&
            draggingPointer.id === event.pointerId
        ) {
            draggingPointer = null;
        }
    }

    function handleWheel(event: WheelEvent): void {
        // handles all mouse wheel and trackpad gestures
        event.preventDefault();

        if (
            event.ctrlKey ||
            (!event.ctrlKey && !zoomWithCtrl)
        ) {
            // zoom with mouse wheel
            // or pinch zoom gesture on trackpads
            scaleOnPoint(
                clamp(mouseZoomCoefficient * event.deltaY, -1 * mouseZoomLimit, mouseZoomLimit),
                event.clientX,
                event.clientY
            );
        } else if (event.altKey) {
            // scroll wheel scrolling
            // or trackpad panning with two fingers
            // ALT key switches X and Y axis
            translate(
                scrollXCoefficient * event.deltaY,
                scrollYCoefficient * event.deltaX
            );
        } else {
            // scroll wheel scrolling
            // or trackpad panning with two fingers
            translate(
                scrollXCoefficient * event.deltaX,
                scrollYCoefficient * event.deltaY
            );
        }
    }

    /* === LIFECYCLE ========================== */
    onMount(() => {
        handleResize();

        // get 2D context
        const tempContext = canvas.getContext("2d");
        if (!tempContext) {
            console.log("failed to get context");
            return;
        } else {
            context = tempContext;
        }

        // get and load imageData
        canvas.height = height;
        canvas.width = width;
        imageData = context.getImageData(0, 0, width, height);
        renderNewImage();

        // create resizeObserver
        const resizeObserver = new ResizeObserver(handleResize);
        resizeObserver.observe(container);

        return () => {
            // disconnect resizeObserver on destroy
            resizeObserver.disconnect();
        }
	});
</script>



<svelte:window on:pointermove={handleMove} />

<div
    class="container"
    bind:this={container}
    on:wheel={handleWheel}
    on:pointerenter={handleEnter}
    on:pointerdown={handleDown}
    on:pointerup={handleUp}
    on:pointercancel={handleCancel}>
    <div
        class="scalor"
        style="transform: scale({scale}) translate({translateX / scale}px, {translateY / scale}px);">
        <canvas bind:this={canvas}></canvas>
    </div>
</div>



<style lang="scss">
    .container {
        display: flex;
        flex-flow: row nowrap;
        align-items: center;
        justify-content: center;
        width: 100%;
        height: 100vh;

        touch-action: none;
        overflow: hidden;
    }

    .scalor {
        transform-origin: center;
        image-rendering: pixelated;
    }

    canvas {
        display: block;
    }
</style>