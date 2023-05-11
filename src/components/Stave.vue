<script>
import {defineComponent, transformVNodeArgs} from 'vue'

const naturalNotes = [
    {name: "C", equalIndex: 0, naturalIndex: 0},
    {name: "D", equalIndex: 2, naturalIndex: 1},
    {name: "E", equalIndex: 4, naturalIndex: 2},
    {name: "F", equalIndex: 5, naturalIndex: 3},
    {name: "G", equalIndex: 7, naturalIndex: 4},
    {name: "A", equalIndex: 9, naturalIndex: 5},
    {name: "B", equalIndex: 11, naturalIndex: 6}
]

function noteToNaturalIndex(noteName) {
    let note = naturalNotes.find(item => noteName.toUpperCase().startsWith(item.name));
    let registry = parseInt(noteName[noteName.length - 1]) - 1
    return registry * 7 + note.naturalIndex
}

function noteToEqualIndex(noteName) {
    let note = naturalNotes.find(item => noteName.toUpperCase().startsWith(item.name));
    let equalIndex = note.equalIndex

    for (let char of noteName.substring(1)) {
        if (char === "b") {
            equalIndex--
        } else if (char === "#") {
            equalIndex++
        } else {
            let registry = parseInt(char) - 1
            if (!isNaN(registry)) {
                return registry * 12 + equalIndex // Abs index
            }
        }
    }

    return equalIndex // Relative index
}

/**
 * Convert equal index to note name. Eg: 0 -> c0, 1 -> c#0 | d♭0, 12 -> c1
 * @param equalIndex
 * @param sharp Whether to use sharp or flat, if equal note does not match natural note
 */
function equalIndexToNote(equalIndex, sharp) {
    let registry = Math.floor(equalIndex / 12)
    if (sharp) {
        let naturalNote;
        for (let i = 0; i < naturalNotes.length; i++) {
            let note = naturalNotes[i]
            if (equalIndex % 12 >= note.equalIndex) {
                naturalNote = note
            } else {
                break
            }
        }
        let note = naturalNote.name
        for (let i = 0; i < equalIndex % 12 - naturalNote.equalIndex; i++) {
            note += "#"
        }
        return note + (registry + 1)
    } else {
        let naturalNote;
        for (let i = 0; i < naturalNotes.length; i++) {
            let note = naturalNotes[i]
            if (equalIndex % 12 <= note.equalIndex) {
                naturalNote = note
                break
            }
        }
        let note = naturalNote.name
        for (let i = 0; i < naturalNote.equalIndex - equalIndex % 12; i++) {
            note += "b"
        }
        return note + (registry + 1)
    }
}

export default defineComponent({
    name: "Stave",
    data() {
        return {
            loading: true,
            notes: [],
            trainingOptions: {
                lowest: "A3",
                highest: "B5",
                count: 24,
            },
            answer: {
                startTime: null,
                cursor: 0,
                correct: 0,
                wrong: 0,
                summary: "",
                status: [],
                lastAnswerTime: null,
            },
            status: {
                totalSpend: 0,
                lastSpend: 0
            },
            keyboard: {
                keys: []
            }
        }
    },
    computed: {
        naturalNotes() {
            return naturalNotes
        }
    },
    mounted() {
        let font = new FontFace("Leland", "url(Leland.otf)")
        document.fonts.add(font)
        font.load().then(() => {
            this.loading = false
            this.resize()
        })
        this.resize()
        window.onresize = () => {
            this.resize()
        }
        window.onkeydown = (key) => {
            this.onKeyEnter(key.key)
        }
        this.generateKeyboard()
    },
    watch: {
        notes: {
            handler() {
                this.render()
            },
            deep: true
        },
        answer: {
            handler() {
                this.render()
            },
            deep: true
        }
    },
    methods: {
        resize() {
            let canvas = this.$refs.canvas
            let container = this.$refs.canvasContainer
            canvas.width = container.clientWidth
            canvas.height = container.clientHeight
            this.render()
        },
        render() {
            let canvas = this.$refs.canvas
            let context = this.$refs.canvas.getContext('2d')
            if (this.loading) return
            context.clearRect(0, 0, canvas.width, canvas.height)
            context.font = "normal 32px \"Leland\", sans-serif"
            context.fillStyle = "rgb(221,221,221)"

            let paddingTop = 50
            let lineHeight = 10
            let lineThickness = 1
            let pxPerIndex = (lineHeight / 2)

            let topPitch = noteToNaturalIndex("f5")
            let bottomPitch = noteToNaturalIndex("e4")

            context.fillRect(0, paddingTop, 1, lineHeight * 4)
            for (let i = 0; i < 5; i++) {
                context.fillRect(0, (paddingTop + lineHeight * i) - lineThickness, canvas.width, lineThickness)
            }
            context.fillRect(canvas.width - 1, paddingTop, 1, lineHeight * 4)

            let puhaoY = (topPitch - noteToNaturalIndex("g4")) * pxPerIndex
            context.fillText("\uE050", 6, paddingTop + puhaoY)

            let offsetX = 40;
            for (let i = 0; i < this.notes.length; i++) {
                if (i > 0 && i % 4 === 0) {
                    // Draw divider
                    context.fillRect(offsetX + 16, paddingTop, 1, lineHeight * 4)
                    offsetX += 16 + 32
                }

                let note = this.notes[i]
                let pitch = noteToNaturalIndex(note.name)
                let noteHeight = 0

                // F6 = 0
                // Draw external line
                if (pitch > topPitch + 1) {
                    for (let j = 0; j < (pitch - topPitch - 1) / 2; j++) {
                        context.fillRect(offsetX - 2, (paddingTop - lineHeight * (j + 1)) - lineThickness, 15, lineThickness)
                    }
                } else if (pitch < bottomPitch - 1) {
                    for (let j = 0; j < (bottomPitch - pitch - 1) / 2; j++) {
                        context.fillRect(offsetX - 2, (paddingTop + lineHeight * (j + 5)) - lineThickness, 15, lineThickness)
                    }
                }

                if (i < this.answer.cursor) {
                    if (this.answer.status[i].correct) {
                        context.fillStyle = "rgb(0,168,56)"
                    } else {
                        context.fillStyle = "rgb(255,78,78)"
                    }
                } else if (i === this.answer.cursor) {
                    context.fillStyle = "rgb(255,221,122)"
                }

                let y = (topPitch - pitch) * pxPerIndex - noteHeight

                // Draw note
                context.fillText("\uE1D5", offsetX, paddingTop + y, 100)
                context.fillStyle = "rgb(221,221,221)"

                offsetX += 32;
            }
        },
        randomNotation() {
            let newNotes = []
            for (let i = 0; i < parseInt(this.trainingOptions.count); i++) {
                let lowest = noteToEqualIndex(this.trainingOptions.lowest)
                let highest = noteToEqualIndex(this.trainingOptions.highest)

                let equalIndex = Math.round(Math.random() * (highest - lowest)) + lowest
                let newNote = equalIndexToNote(equalIndex, true)
                newNotes.push({name: newNote, duration: "8"})
            }
            this.answer = {
                startTime: new Date().getTime(),
                cursor: 0,
                correct: 0,
                wrong: 0,
                summary: "",
                status: [],
                lastAnswerTime: new Date().getTime()
            }
            this.notes = newNotes
        },
        generateKeyboard() {
            let keys = []
            for (let i = 3; i < 6; i++) {
                naturalNotes.forEach(item => {
                    keys.push({
                        name: item.name,
                        nameWithRegistry: item.name + i,
                        naturalIndex: noteToNaturalIndex(item.name + i),
                        relativeNaturalIndex: noteToNaturalIndex(item.name + i) % 7
                    })
                })
            }
            this.keyboard.keys = keys
        },
        onKeyEnter(key) {
            let map = {
                "q": "C3",
                "w": "D3",
                "e": "E3",
                "r": "F3",
                "t": "G3",
                "y": "A3",
                "u": "B3",
                "a": "C4",
                "s": "D4",
                "d": "E4",
                "f": "F4",
                "g": "G4",
                "h": "A4",
                "j": "B4",
                "z": "C5",
                "x": "D5",
                "c": "E5",
                "v": "F5",
                "b": "G5",
                "n": "A5",
                "m": "B5",
            }
            let noteNameWithRegistry = map[key]
            this.keyboardClick({
                nameWithRegistry: noteNameWithRegistry,
                naturalIndex: noteToNaturalIndex(noteNameWithRegistry),
            })
        },
        keyboardClick(note) {
            let currentNote = this.notes[this.answer.cursor]
            let currentNaturalIndex = noteToNaturalIndex(currentNote.name)
            let correct = currentNaturalIndex === note.naturalIndex

            if (correct) this.answer.correct++
            else this.answer.wrong++

            this.answer.status[this.answer.cursor] = {
                correct: correct,
                answer: note.nameWithRegistry,
                duration: new Date().getTime() - this.answer.lastAnswerTime
            }
            this.status.totalSpend = new Date().getTime() - this.answer.startTime
            this.status.lastSpend = new Date().getTime() - this.answer.lastAnswerTime
            this.answer.lastAnswerTime = new Date().getTime()
            this.answer.cursor += 1

            // this.notes.push({name: note, duration: "8"})
        }
    }
})
</script>

<template>
    <div class="container">
        <div class="controller">
            <n-button @click="randomNotation" type="primary">
                Start
            </n-button>
            <n-input style="width: fit-content" type="text" v-model:value="trainingOptions.lowest"
                     placeholder="Lowest"></n-input>
            ~
            <n-input style="width: fit-content" type="text" v-model:value="trainingOptions.highest"
                     placeholder="Highest"></n-input>
            <n-input style="width: fit-content" type="text" v-model:value="trainingOptions.count"
                     placeholder="Count"></n-input>
        </div>
        <div ref="canvasContainer" class="canvas-container">
            <canvas ref="canvas" class="canvas">
            </canvas>
        </div>
        <div class="status">
            Total spend: {{status.totalSpend}}ms
            Last spend: {{status.lastSpend}}ms
        </div>
        <div class="keyboard">
            <div :class="{ 'item': true, 'c': item.relativeNaturalIndex === 0, 'middle-c': item.nameWithRegistry === 'C4' }"
                 v-for="item in keyboard.keys" @click="keyboardClick(item)">{{ item.name === 'C' ? item.nameWithRegistry : item.name }}
            </div>
        </div>
    </div>
</template>

<style scoped>
.container {
    display: flex;
    flex-direction: column;
    height: 100%;
    gap: 16px;
    padding: 24px;
}

.controller {
    display: flex;
    flex-direction: row;
    gap: 16px;
}

.canvas-container {
    width: 100%;
    flex-grow: 1;
    display: flex;
    padding: 0;
}

.canvas {
    position: absolute;
    font: normal 30px "Leland", sans-serif;
}

.keyboard {
    display: flex;
    flex-direction: row;
    align-self: center;
    gap: 2px;

    .item {
        display: flex;
        border-radius: 4px;
        align-items: center;
        justify-content: center;
        width: 50px;
        height: 80px;
        background: white;
        user-select: none;
        color: black;
    }

    .c {
        background: #9e9e9e;
    }

    .middle-c {
        background: #fff2a2;
    }
}
</style>