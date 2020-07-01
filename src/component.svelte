<style>
    button {
        @apply font-bold py-2 px-4 rounded;
        @apply bg-blue-500 text-white;
    }

    button:disabled {
        @apply bg-gray-700;
        @apply cursor-not-allowed;
    }

    button:hover:enabled {
        @apply bg-blue-700;
    }

    button:focus {
        @apply outline-none;
    }

    input {
        @apply w-full;
    }
</style>

<script>
    import * as workerTimers from "worker-timers";
    import {get, writable} from 'svelte/store';

    let minDurationSeconds = writable(180);
    let maxDurationSeconds = writable(900);
    let strokesPerMinute = 100;
    let volume = 0.8;
    let counter = 0;
    let countDown = 4;
    let minFormattedDuration = writable("");
    let maxFormattedDuration = writable("");
    let targetCount = 0;

    let intervalId;
    let running = false;
    let testing = false;

    const cum = new Howl({
        src: ["assets/cum.mp3"]
    });

    const synth = new Tone.Synth({
        oscillator: {
            type: 'triangle',
            modulationType: 'sine',
            modulationIndex: 3,
            harmonicity: 3.4
        },
        envelope: {
            attack: 0.001,
            decay: 0.1,
            sustain: 0.01,
            release: 0.1
        }
    }).toDestination();

    const playBeat = () => {
        synth.triggerAttackRelease('A1', '16n');
    }

    const playBeep = () => {
        synth.triggerAttackRelease('C3', '4n');
    }

    const playStartBeep = () => {
        synth.triggerAttackRelease('C5', '4n');
    }

    const start = () => {
        running = true;
        const targetSeconds = Math.random() * ($maxDurationSeconds - $minDurationSeconds) + $minDurationSeconds;
        targetCount = Math.round(targetSeconds * strokesPerMinute / 60);
        intervalId = workerTimers.setInterval(function beatLoop() {
            if (countDown > 0) {
                countDown = countDown - 1;
            } else {
                counter = counter + 1;
            }
            if (counter === targetCount) {
                cum.play();
                stop();
            } else {
                if (countDown > 0) {
                    playBeep();
                } else if (countDown === 0 && counter === 0) {
                    playStartBeep();
                } else {
                    playBeat();
                }
            }
            return beatLoop;
        }(), 60 / strokesPerMinute * 1000);
    }

    const stop = () => {
        running = false;
        counter = 0;
        countDown = 4;
        workerTimers.clearInterval(intervalId);
    }

    const restart = () => {
        if (running) {
            stop();
            start();
        }
    }

    const test = () => {
        testing = true;
        targetCount = 3
        intervalId = workerTimers.setInterval(function testBeatLoop() {
            counter = counter + 1;
            playBeat();
            if (counter === targetCount) {
                testing = false;
                stop();
            }
            return testBeatLoop;
        }(), 60 / strokesPerMinute * 1000);
    }

    $: if (strokesPerMinute) {
        restart();
    }

    $: if (volume) {
        Howler.volume(volume);
    }

    minDurationSeconds.subscribe(() => {
        if ($minDurationSeconds <= 60) {
            minFormattedDuration.set(`${$minDurationSeconds} seconds`);
        } else {
            const seconds = ($minDurationSeconds % 60).toString().padStart(2, "0");
            minFormattedDuration.set(`${Math.floor($minDurationSeconds / 60)}:${seconds} minutes`);
        }
    });

    $: if ($minDurationSeconds > get(maxDurationSeconds)) {
        maxDurationSeconds.set($minDurationSeconds);
    }

    maxDurationSeconds.subscribe(() => {
        if ($maxDurationSeconds <= 60) {
            maxFormattedDuration.set(`${$maxDurationSeconds} seconds`);
        } else {
            const seconds = ($maxDurationSeconds % 60).toString().padStart(2, "0");
            maxFormattedDuration.set(`${Math.floor($maxDurationSeconds / 60)}:${seconds} minutes`);
        }
    });

    $: if ($maxDurationSeconds < get(minDurationSeconds)) {
        minDurationSeconds.set($maxDurationSeconds);
    }


</script>

<div class="container mx-auto px-4 py-4">
    <div class="grid grid-flow-row gap-4">
        <div>
            {#if !running}
                <button on:click={start} disabled={testing}>Start</button>
                <button on:click={test} disabled={testing}>Test (Play 3 beats)</button>
            {:else}
                <button on:click={stop}>Stop</button>
            {/if}
        </div>
        {#if running}
            {#if countDown > 0}
                <div>
                    Ready? {countDown}
                </div>
            {:else}
                <div class="font-mono">
                    Stokes: {counter}
                </div>
            {/if}
        {/if}
        {#if !running}
            <div class="grid grid-cols-6 gap-4">
                <div class="col-span-5">
                    <label>Minimum Duration
                        <input step="15" min="15" max="1800" type="range" bind:value={$minDurationSeconds}>
                    </label>
                </div>
                <div class="col-span-1 flex pb-1">
                    <output class="flex self-end">{$minFormattedDuration}</output>
                </div>
            </div>
            <div class="grid grid-cols-6 gap-4">
                <div class="col-span-5">
                    <label>Maximum Duration
                        <input step="15" min="15" max="1800" type="range" bind:value={$maxDurationSeconds}>
                    </label>
                </div>
                <div class="col-span-1 flex pb-1">
                    <output class="flex self-end">{$maxFormattedDuration}</output>
                </div>
            </div>
            <div class="grid grid-cols-6 gap-4">
                <div class="col-span-5">
                    <label>Strokes per Minute
                        <input step="10" min="10" max="400" type="range" bind:value={strokesPerMinute}>
                    </label>
                </div>
                <div class="col-span-1 flex pb-1">
                    <output class="flex self-end">{strokesPerMinute}</output>
                </div>
            </div>
        {/if}
        <div class="grid grid-cols-6 gap-4">
            <div class="col-span-5">
                <label>Volume
                    <input step="0.1" min="0" max="1" type="range" bind:value={volume}>
                </label>
            </div>
            <div class="col-span-1 flex pb-1">
                <output class="flex self-end">{volume * 100}%</output>
            </div>
        </div>
    </div>
</div>