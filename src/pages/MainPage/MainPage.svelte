<script lang="ts">
    import styles from './main-page.module.scss'
    
    import { onMount } from 'svelte'
    import { writable } from 'svelte/store'
    import * as Tone from 'tone'

    const WAVEFORM_COLORS = ['#00FFFF', '#FFD700', '#FF69B4']
    const WAVEFOTM_LINE_WIDTH = [4, 2, 1]

    const drawWaveforms = () => {
        const canvas = document.getElementById('wave') as HTMLCanvasElement
        const ctx = canvas.getContext('2d')

        if (!ctx) {
            console.error('Failed to get canvas context')
            return
        }

        requestAnimationFrame(drawWaveforms)

        ctx.clearRect(0, 0, canvas.width, canvas.height);

        Object.values($soundStore).forEach((item, waveIndex) => {
            const { analyser } = item
            if (!analyser) return

            const bufferLength = analyser.size
            const data = analyser.getValue()

            ctx.beginPath()
            const sliceWidth = canvas.width / bufferLength
            let x = 0;

            for (let i = 0; i < bufferLength; i++) {
                const v = data[i]
                const y = (1 - Number(v)) * canvas.height / 2
                ctx.lineTo(x, y)
                x += sliceWidth
            }

            ctx.strokeStyle = WAVEFORM_COLORS[waveIndex]
            ctx.lineWidth = WAVEFOTM_LINE_WIDTH[waveIndex]
            ctx.stroke()
        })
    }

    const getTimeItems = () => {
        const time = new Date()
        const timeItems = [time.getHours(), time.getMinutes(), time.getSeconds()]
        return timeItems.map((item) => item.toString().padStart(2, '0'))
    }

    const timeItems = getTimeItems()
    let hours = timeItems[0]
    let minutes = timeItems[1]
    let seconds = timeItems[2]

    function createFrequencyGroup(minFreq: number, maxFreq: number, count: number) {
      return Array.from({ length: count }, (_, i) => {
        const t = i / (count - 1)
        return minFreq * Math.pow(maxFreq / minFreq, t)
      });
    }
    const LOW = createFrequencyGroup(20, 199, 24);   // for hours
    const MID = createFrequencyGroup(200, 799, 60);  // for minutes
    const HIGH = createFrequencyGroup(800, 2599, 60); // for seconds

    let isStarted = false
    let currentFrequencies: Array<number> = []

    interface SoundStoreItem {
        synth: Tone.Oscillator | null
        analyser: Tone.Analyser | null
        index: number | null
    }

    interface SoundStore {
        hour: SoundStoreItem
        minute: SoundStoreItem
        second: SoundStoreItem
    }

    const storeDefaultState: SoundStore = {
        hour: {
            synth: null,
            analyser: null,
            index: null,
        },
        minute: {
            synth: null,
            analyser: null,
            index: null,
        },
        second: {
            synth: null,
            analyser: null,
            index: null,
        }
    }

    let soundStore = writable({ ...storeDefaultState })

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

        if (!$soundStore[name].analyser) {
            const currentAnalyser = new Tone.Analyser('waveform', 128)
            
            const currentSynth = $soundStore[name].synth
            if (currentSynth) {
                currentSynth.connect(currentAnalyser)
            }
            
            soundStore.update((state) => ({
                ...state,
                [name]: {
                    ...state[name],
                    analyser: currentAnalyser
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
        const now = Tone.now()

        const time = new Date()
        const [h, m, s] = [time.getHours(), time.getMinutes(), time.getSeconds()]
        currentFrequencies = [LOW[h], MID[m], HIGH[s]]

        handleSoundItem('hour', LOW[h], h, now)
        handleSoundItem('minute', MID[m], m, now)
        handleSoundItem('second', HIGH[s], s, now)
    }

    onMount(() => {
        drawWaveforms()

        const interval = setInterval(() => {
            const timeItems = getTimeItems()
            hours = timeItems[0]
            minutes = timeItems[1]
            seconds = timeItems[2]

            //console.log(`🕒 ${hours}:${minutes}:${seconds}`)

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

    // $:{
    //     console.log('Sound Store:', $soundStore)
    // }

    $: timer = `${hours}:${minutes}:${seconds}`
</script>

<svelte:head>
    <title>{timer}</title> 
</svelte:head>

<div class={styles.wrapper}>
    <canvas class={styles.canvas} id="wave"></canvas>

    <div class={styles.clockWrapper}>
        <span class={styles.clock}>{timer}</span>

        {#if !isStarted}
            <button class={styles.button} on:click={startSynth}>Start</button>
        {:else}
            <button class={styles.button} on:click={stopSynth}>Stop</button>
        {/if}

        <!-- {#each currentFrequencies as freq}
            <div>{freq.toFixed(1)} Hz</div>
        {/each} -->
    </div>
</div>