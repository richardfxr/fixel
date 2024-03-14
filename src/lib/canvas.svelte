<script lang="ts">
    /* === IMPORTS ============================ */
    import { onMount } from 'svelte';

    /* === TYPES ============================== */
    type Pointer = {
        id: number,
        x: number,
        y: number
    };

    /* === CONSTANTS ========================== */
    const panXSensitivity = 1;
    const panYSensitivity = 1;
    const pinchZoomSensitivity = 0.23;
    const mouseZoomCoefficient = -1;
    const mouseZoomLimit = 7;
    const scrollXCoefficient = 0.9;
    const scrollYCoefficient = 0.9;
    const zoomWithCtrl = true;
    const ongoingPointers: Pointer[] = [];
    const imageWidth = 6;
    const imageHeight = 6;
    const scaleMin = 10;
    const scaleMax = 100;
    const image = new Uint8ClampedArray([
        0, 0, 0, 255,
        51, 0, 0, 255,
        102, 0, 0, 255,
        153, 0, 0, 255,
        204, 0, 0, 255,
        255, 0, 0, 255,
        0, 51, 0, 255,
        51, 51, 0, 255,
        102, 51, 0, 255,
        153, 51, 0, 255,
        204, 51, 0, 255,
        255, 51, 0, 255,
        0, 102, 0, 255,
        51, 102, 0, 255,
        102, 102, 0, 255,
        153, 102, 0, 255,
        204, 102, 0, 255,
        255, 102, 0, 255,
        0, 153, 0, 255,
        51, 153, 0, 255,
        102, 153, 0, 255,
        153, 153, 0, 255,
        204, 153, 0, 255,
        255, 153, 0, 255,
        0, 204, 0, 255,
        51, 204, 0, 255,
        102, 204, 0, 255,
        153, 204, 0, 255,
        204, 204, 0, 255,
        255, 204, 0, 255,
        0, 255, 0, 255,
        51, 255, 0, 255,
        102, 255, 0, 255,
        153, 255, 0, 255,
        204, 255, 0, 255,
        255, 255, 0, 255,
    ]);

    /* === VARIABLES ========================== */
    let width = 100;
    let height = 100;
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
            -1 * (width / 2 - 1),
            width / 2 - 1
        );
        translateY = clamp(
            translateY + translateYDiff,
            -1 * (height / 2 - 1),
            height / 2 - 1
        );
    }

    function scaleOnPoint(scaleDiff: number, pointX: number, pointY: number): void {
        const newScale = clamp(scale + scaleDiff, scaleMin, scaleMax);

        // adds addtional translates so that the scale transform appear to be centered
        // on point (pointX, pointY)
        translate(
            -1 * ((pointX - width / 2) - translateX) / scale * (newScale - scale),
            -1 * ((pointY - height / 2) - translateY) / scale * (newScale - scale)
        );

        scale = newScale
    }

    /* === EVENT HANDLES ====================== */
    function handleResize(): void {
        // update width and height
        width = container.clientWidth;
        height = container.clientHeight;

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

    function handleUp(event: PointerEvent): void {
        if (
            draggingPointer &&
            event.pointerId === draggingPointer.id
        ) {
            // the pointer was dragging, update translates (position)
            translate(
                panXSensitivity * (event.clientX - draggingPointer.x),
                panYSensitivity * (event.clientY - draggingPointer.y)
            );
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
        canvas.height = imageHeight;
        canvas.width = imageWidth;
        imageData = context.getImageData(0, 0, imageWidth, imageHeight);
        imageData.data.set(image);
        context.putImageData(imageData, 0, 0);

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