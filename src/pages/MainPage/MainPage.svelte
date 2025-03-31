<script>
    import { onMount } from 'svelte'
    import * as Tone from 'tone'
    import dayjs from 'dayjs'

    const now = dayjs()
    let hours = now.format('HH')
    let minutes = now.format('mm')
    let seconds = now.format('ss')

    onMount(() => {
        const interval = setInterval(() => {
            const now = dayjs()
            hours = now.format('HH')
            minutes = now.format('mm')
            seconds = now.format('ss')
        }, 1000)

        return () => clearInterval(interval)
    })

    const handleStartTone = async () => {
        const synth = new Tone.Synth().toDestination()
        synth.triggerAttackRelease('C4', '8n')
    }
</script>

<h1>Kāla</h1>

<h2>{`${hours}:${minutes}:${seconds}`}</h2>

<button on:click={handleStartTone}>Start</button>