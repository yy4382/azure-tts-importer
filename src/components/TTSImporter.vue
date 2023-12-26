<template>
    <div class="tts-importer">
        <div id="key-form">
            <div id="key-region">
                <div class="key-input">
                    <label for="api_key">API Key: </label>
                    <input type="text" id="api_key" v-model="apiKey" />
                </div>
                <div class="key-input">
                    <label for="api_region">API Region: </label>
                    <input type="text" id="api_region" v-model="apiRegion" />
                </div>
            </div>
            <button @click="getVoiceList">提交</button>
        </div>
        <div style="display:flex; flex-direction: column; align-items: self-start;">
            <div>
                <label for="voiceSelect">声音 (voice)：</label>
                <select id="voiceSelect" v-model="selectedVoice">
                    <option v-for="item in voiceList" :key="item.ShortName" :value="item">
                        {{ item.LocalName + " " + item.ShortName }}
                    </option>
                </select>
            </div>
            <div>
                <div style="display: flex;flex-direction: row; align-items: self-start;">
                    <input type="checkbox" id="useVoiceStyle" v-model="useVoiceStyle" :disabled="!hasStyle" />
                    <label for="useVoiceStyle" v-if="hasStyle">使用声音风格</label>
                    <label for="useVoiceStyle" v-else>使用声音风格（禁用，该声音没有风格）</label>
                </div>
                <div v-if="useVoiceStyle">
                    <label style="align-items: self-start;" for="voiceStyleSelect">声音风格 (voiceStyle)：</label>
                    <select id="voiceStyleSelect" v-model="selectedVoiceStyle">
                        <option v-for="style in selectedVoice.VoiceStyleNames" :key="style" :value="style">{{ style }}
                        </option>
                    </select>
                </div>
            </div>
            <div>
                <label for="rateRange">语速 (rate)：</label>
                <select name="sudo" id="rateRange" v-model="selectedRate">
                    <option value="default">默认 (default)</option>
                    <option value="x-slow">极低 (x-slow)</option>
                    <option value="slow">低 (slow)</option>
                    <option value="medium">中 (medium)</option>
                    <option value="fast">高 (fast)</option>
                    <option value="x-fast">极高 (x-fast)</option>
                </select>
            </div>
            <div>
                <label for="pitchRange">音调(pitch)：</label>
                <select name="sudo" id="pitchRange" v-model="selectedPitch">
                    <option value="default">默认 (default)</option>
                    <option value="x-low">极低 (x-low)</option>
                    <option value="low">低 (low)</option>
                    <option value="medium">中 (medium)</option>
                    <option value="high">高 (high)</option>
                    <option value="x-high">极高 (x-high)</option>
                </select>
            </div>
        </div>
    </div>
    <button @click="copyLegadoConfig">复制配置</button>
</template>

<script>
import axios from 'axios'
export default {
    name: 'TTSImporter',
    data() {
        return {
            apiKey: '',
            apiRegion: 'eastasia',
            voiceList: [],
            selectedVoice: {},
            hasStyle: false,
            useVoiceStyle: false,
            selectedVoiceStyle: null,
            selectedRate: 'default',
            selectedPitch: 'default',
        }
    },
    created() {
        this.apiKey = localStorage.getItem('apiKey') || ''
        this.apiRegion = localStorage.getItem('apiRegion') || 'eastasia'
        this.voiceList = JSON.parse(localStorage.getItem('voiceList')) || []
    },
    watch: {
        apiKey(newVal) {
            localStorage.setItem('apiKey', newVal)
        },
        apiRegion(newVal) {
            localStorage.setItem('apiRegion', newVal)
        },
        voiceList(newVal) {
            if (newVal.length > 0) {
                localStorage.setItem('voiceList', JSON.stringify(newVal))
            }
        },
        selectedVoice(newVal) {
            console.log(newVal.VoiceStyleNames)
            if (!newVal.VoiceStyleNames || newVal.VoiceStyleNames.length === 0 || newVal.VoiceStyleNames[0] === "") {
                console.log("no style")
                this.hasStyle = false
                this.useVoiceStyle = false
                this.selectedVoiceStyle = null
            } else {
                console.log("has style")
                this.hasStyle = true
                // this.useVoiceStyle = true
                // this.selectedVoiceStyle = newVal.VoiceStyleNames[0]
            }
        },
        useVoiceStyle(newVal) {
            if (newVal) {
                this.selectedVoiceStyle = this.selectedVoice.VoiceStyleNames[0]
            } else {
                this.selectedVoiceStyle = null
            }
        }
    },
    methods: {
        getVoiceList() {
            console.log(this.apiKey, this.apiRegion)
            axios.get(`https://eastasia.tts.speech.microsoft.com/cognitiveservices/voices/list`, {
                headers: {
                    'Ocp-Apim-Subscription-Key': this.apiKey,
                }
            }).then(res => {
                // console.log(res.data)
                const zhVoices = res.data.filter(voice =>
                    voice.Locale.startsWith("zh")
                ).map(voice => {
                    const styles = voice.StyleList || [""];
                    return {
                        LocalName: voice.LocalName,
                        ShortName: voice.ShortName,
                        VoiceStyleNames: styles,
                    };
                });
                this.voiceList = zhVoices
                console.log(this.voiceList)
            }).catch(err => {
                console.log(err)
            })
        },
        generateLegadoConfig() {
            let header = {
                "Ocp-Apim-Subscription-Key": this.apiKey,
                "Content-Type": "application/ssml+xml",
                "X-Microsoft-OutputFormat": "audio-16khz-128kbitrate-mono-mp3",
                "User-Agent": "legado"
            }
            let ssml = `
            <speak version="1.0" xml:lang="zh-CN">
                <voice name="${this.selectedVoice.ShortName}">
                    <prosody rate="{{speakSpeed*4}}%" pitch="${this.selectedPitch}">
                        ${this.useVoiceStyle ? `<mstts:express-as style="${this.selectedVoiceStyle}">` : ""}
                        {{speakText}}
                        ${this.useVoiceStyle ? `</mstts:express-as>` : ""}
                    </prosody>
                </voice>
            </speak>`
            let urlConfig = {
                method: "POST",
                body: ssml
            }
            let config = {
                "concurrentRate": "0",
                "contentType": "audio/mpeg",
                "header": JSON.stringify(header),
                "id": 1642068899362,
                "loginCheckJs": "",
                "loginUi": "",
                "loginUrl": "",
                "name": `Azure ${this.selectedVoice.LocalName}${this.selectedVoiceStyle || ""}${this.selectedPitch === "default" ? "" : " - " + this.selectedPitch}`,
                "url": `https://${self.apiRegion}.tts.speech.microsoft.com/cognitiveservices/v1,${JSON.stringify(urlConfig)}`
            }
            return JSON.stringify(config)
        },
        copyLegadoConfig() {
            let config = this.generateLegadoConfig()
            try {
                navigator.clipboard.writeText(config);
            } catch (err) {
                console.log(err)
                alert("复制失败，请手动复制")
            }
        },
    }
}
</script>
<style>
.tts-importer {
    display: flex;
    flex-direction: column;
    margin: 0 3vw;
}

#key-form {
    display: flex;
    flex-direction: row;
    justify-content: center;
}

#key-region {
    display: flex;
    flex-direction: column;
    margin: 1em 0;
    padding: 0 1em;
    background-color: aquamarine;
    border-radius: 1rem;
}

.key-input {
    display: flex;
    flex-direction: row;
    align-items: start;
    margin-right: 1em;
    padding: 3px 0;
}

button {
    margin-left: 5px;
    margin-right: 5px;
    margin-top: 10px;
    margin-bottom: 10px;
    padding: 5px;
    border-radius: 5px;
    background-color: #fff;
    box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}
</style>