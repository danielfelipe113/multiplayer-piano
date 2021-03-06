<template>
    <div class="piano-key" :class="{'is-black-key': note.isBlackKey, 'is-playing': note.playing}">
        <div @mouseout="onMouseOut" v-for="user in note.users" :key="user.id" class="piano-key-user" :style="{background: user.color, boxShadow: `0 0 50px ${user.color}`}"></div>
    </div>

</template>

<script>
    import {mapGetters, mapActions, mapState, mapMutations} from 'vuex'
    import Note from '../midi/Note'
    import {MIDI_ATTACK, MIDI_RELASE, SOURCE_MIDI, SOURCE_MOUSE} from "../midi/constants";
    import {
        CLEAR_PIANO_PLAYING,
        USER_PLAY_NOTE,
        USER_RELEASE_NOTE,
        USER_RELEASE_SUSTAIN
    } from "../../store/modules/piano/actions";
    import {DELETE_ALL_KEYS_DOWN} from "../../store/modules/piano/mutations";
    import AudioEngine from '../../components/audio/AudioEngine'
    import { EventBus } from "../../main";
    const MAX_VELOCITY = 1
    const VELOCITY_STEPS = 127

    export default {
        name: 'piano-key',
        props: {
            note: {type: Note, required: true},
            sustain: {type: Boolean, required: false}
        },
        computed: {
            currentUserPlaying() {
                return !!this.note.users.find(user => this.currentUser.id === user.id)
            },
            ...mapState('users', [
                'currentUser'
            ]),
            ...mapState('piano', [
                'canPlay'
            ]),
            ...mapGetters('piano', [
                'midiAccess'
            ])
        },
        methods: {
            ...mapMutations('piano', [
                DELETE_ALL_KEYS_DOWN
            ]),
            play(volume, source = SOURCE_MOUSE) {
                if (!this.canPlay) return;
                this[USER_PLAY_NOTE]({
                    note: this.note,
                    volume,
                    source
                })
            },
            release(delay = 3) {
                if (!this.canPlay) return;
                this[USER_RELEASE_NOTE]({
                    note: this.note,
                    delay,
                    sustained: this.sustain
                })
            },
            blur() {
                if (this.currentUserPlaying) {
                    this[CLEAR_PIANO_PLAYING]()
                }
                if (this.midiAccess) {
                    this.midiAccess.stopListening()
                }
            },
            focus () {
                if (this.midiAccess) {
                    this.midiAccess.startListening()
                }
            },
            onMouseEnter (e) {
                if (e.buttons > 0 && !this.currentUserPlaying) {
                    EventBus.$emit('note.release')
                    this.play(0.5)
                }
            },
            onMouseDown() {
                if (AudioEngine.state === 'suspended') {
                    AudioEngine.resume();
                }
                if (!this.currentUserPlaying) this.play(0.5)
            },
            onMouseOut() {
                this.release()
            },
            onMouseUp() {
                this.release()
            },
            ...mapActions('piano', [
                USER_PLAY_NOTE,
                USER_RELEASE_NOTE,
                USER_RELEASE_SUSTAIN,
                CLEAR_PIANO_PLAYING
            ])
        },
        created() {
            if (this.midiAccess) {
                this.midiAccess.startListening()
                this.midiAccess.listenToMidiForNote(MIDI_ATTACK, this.note, (e) => {
                    if (AudioEngine.state === 'suspended') {
                        EventBus.$emit('midi.firstUserInteraction')
                    }
                    this.play(e.velocity * (MAX_VELOCITY / VELOCITY_STEPS), SOURCE_MIDI)
                })
                this.midiAccess.listenToMidiForNote(MIDI_RELASE, this.note, () => {
                    this.release(3)
                })
            }
        },
        mounted() {
            this.onMouseDown = this.onMouseDown.bind(this)
            this.onMouseUp = this.onMouseUp.bind(this)
            this.onMouseOut = this.onMouseOut.bind(this)
            this.onMouseEnter = this.onMouseEnter.bind(this)
            this.blur = this.blur.bind(this)
            this.focus = this.focus.bind(this)
            EventBus.$on('note.release', () => {
                if (this.note.source === SOURCE_MOUSE && this.currentUserPlaying) {
                    this.release()
                }
            })
            window.addEventListener('blur', this.blur)
            window.addEventListener('focus', this.focus)
            window.addEventListener('contextmenu', this.blur)
            this.$el.addEventListener('mousedown', this.onMouseDown)
            this.$el.addEventListener('mouseup', this.onMouseUp)
            this.$el.addEventListener('mouseenter', this.onMouseEnter)
            // this.$el.addEventListener('mouseout', this.onMouseOut)
        },
        beforeDestroy() {
            if (this.midiAccess)
                this.midiAccess.stopListening()
            window.removeEventListener('blur', this.blur)
            window.removeEventListener('focus', this.focus)
            window.removeEventListener('contextmenu', this.blur)
            this.$el.removeEventListener('mousedown', this.onMouseDown)
            this.$el.removeEventListener('mouseup', this.onMouseUp)
            // this.$el.removeEventListener('mouseout', this.onMouseOut)
        }
    }
</script>

<style lang="scss">
    @import "../../assets/scss/modules/variables";

    .piano-key {
        background: whitesmoke;
        height: 140px;
        width: 30px;
        border-right: $piano-key-border;
        -webkit-user-drag: none;
        &:first-of-type {
            border-left: none;
        }
        &:last-of-type {
            border-right: none;
        }
        position: relative;
        &.is-black-key {
            background: $black-key-color;
            width: 20px;
            height: 100px;
            border: $piano-key-border;
        }
        & .piano-key-user {
            position: absolute;
            content: '';
            width: 100%;
            height: 100%;
        }
        &.is-playing {
            @each $colorName, $color in $color-list {
                &.#{$colorName} {
                    background: $color
                }
            }
        }
    }


</style>