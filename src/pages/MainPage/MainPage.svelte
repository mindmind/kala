<script lang="ts">
    import { onMount } from 'svelte'
    import { writable } from 'svelte/store'
    import * as Tone from 'tone'

    const getTimeItems = () => {
        const time = new Date();
        const timeItems = [time.getHours(), time.getMinutes(), time.getSeconds()];
        return timeItems.map((item) => item.toString().padStart(2, '0'))
    }

    const timeItems = getTimeItems()
    let hours = timeItems[0]
    let minutes = timeItems[1]
    let seconds = timeItems[2]

    function createFrequencyGroup(minFreq: number, maxFreq: number, count: number) {
      return Array.from({ length: count }, (_, i) => {
        const t = i / (count - 1);
        return minFreq * Math.pow(maxFreq / minFreq, t);
      });
    }
    const LOW = createFrequencyGroup(20, 200, 24);   // for hours
    const MID = createFrequencyGroup(200, 800, 60);  // for minutes
    const HIGH = createFrequencyGroup(800, 2600, 60); // for seconds

    let isStarted = false
    let currentFrequencies: Array<number> = [];

    interface SoundStoreItem {
        synth: Tone.Oscillator | null;
        index: number | null;
    }

    interface SoundStore {
        hour: SoundStoreItem;
        minute: SoundStoreItem;
        second: SoundStoreItem;
    }

    const storeDefaultState: SoundStore = {
        hour: {
            synth: null,
            index: null,
        },
        minute: {
            synth: null,
            index: null,
        },
        second: {
            synth: null,
            index: null,
        }
    }

    let soundStore = writable({ ...storeDefaultState });

    const handleSoundItem = (name: keyof typeof storeDefaultState, freq: number, timeItemIndex: number, now: number) => {
        if (!$soundStore[name].synth) {
            const currentSynth = new Tone.Oscillator(freq, 'sine').toDestination()

            if (name === 'second') {
                currentSynth.volume.value = -20
            } else if (name === 'minute') {
                currentSynth.volume.value = -10
            }

            currentSynth.start(now)

            soundStore.update((state) => ({
                ...state,
                [name]: {
                    synth: currentSynth,
                    index: timeItemIndex
                }
            }))
        }

        if ($soundStore[name].synth && $soundStore[name].index !== timeItemIndex) {
            $soundStore[name].synth.set({ frequency: freq })
            soundStore.update((state) => ({
                ...state,
                [name]: {
                    ...state[name],
                    index: timeItemIndex
                }
            }))
        }
    }

    async function play() {
        const now = Tone.now();

        const time = new Date();
        const [h, m, s] = [time.getHours(), time.getMinutes(), time.getSeconds()];
        currentFrequencies = [LOW[h], MID[m], HIGH[s]];

        handleSoundItem('hour', LOW[h], h, now)
        handleSoundItem('minute', MID[m], m, now)
        handleSoundItem('second', HIGH[s], s, now)
    }

    onMount(() => {
        const interval = setInterval(() => {
            const timeItems = getTimeItems()
            hours = timeItems[0]
            minutes = timeItems[1]
            seconds = timeItems[2]

            console.log(`🕒 ${hours}:${minutes}:${seconds}`)

            if (isStarted) {
                play()
            }

        }, 1000)

        return () => clearInterval(interval)
    })

    const startSynth = async () => {
        if (!isStarted) {
            await Tone.start()
            .then(() => {
                isStarted = true
            })
        }
    }

    const stopSynth = () => {
        if (isStarted) {
            soundStore.update((state) => {
                Object.values(state).forEach((item) => {
                    if (item.synth) {
                        item.synth.dispose()
                    }
                });
                return { ...storeDefaultState }
            })

            isStarted = false
        }
    }

    $:{
        console.log('Sound Store:', $soundStore)
    }
</script>

<h1>Kāla</h1>

<h2>{`${hours}:${minutes}:${seconds}`}</h2>

{#if !isStarted}
    <button on:click={startSynth}>Start</button>
{:else}
    <button on:click={stopSynth}>Stop</button>
{/if}

{#each currentFrequencies as freq}
    <div>{freq.toFixed(1)} Hz</div>
{/each}