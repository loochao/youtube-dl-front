<template>
    <div class="d-flex justify-content-center">
        <div style="position: relative;">
            <input placeholder="Paste video url" @blur="getInfo" v-model="options.url" type="url" />
            <button v-if="isDownloading && canDownload" disabled>Downloading..</button>
            <button v-else-if="canDownload && !isDownloading" @click="sendUrl">Download</button>
            <button v-else-if="isFetchingInfo" disabled>Loading..</button>
            <button v-else-if="!canDownload" disabled>Download</button>
        </div>
        <div>
            <button @click="selectAudioOnly" v-if="options.audioOnly && !isDownloading" class="audio-button" id="audio-button-active">Audio</button>
            <button v-else-if="options.audioOnly || !options.audioOnly && isDownloading" id="audio-disabled-button" disabled class="audio-button">Audio</button>
            <button @click="selectAudioOnly" v-if="!options.audioOnly && !isDownloading" class="audio-button">Audio</button>
        </div>
        <select v-if="info && !isDownloading && !options.audioOnly" v-model="options.videoQuality">
            <option disabled selected>Quality</option>
            <option
            v-for="(format, index) in getFormats"
            :key="index"
            :value="format.formatId"
            >{{format.formatNote}}</option>
        </select>
        <select v-else disabled v-model="options.videoQuality">
            <option disabled selected>Quality</option>
        </select>
    </div>
</template>
<script>
import axios from 'axios';

export default {
    name: "Header",
    data: () => {
        return({
            info: null,
            options : {
                playlist: false,
                url : "",
                videoQuality: "Quality",
                soundQuality: null,
                audioOnly: false,
                target: ""
            },
            isDownloading: false,
            canDownload: false,
            isFetchingInfo: false,
        });
    },
    methods: {
        sendUrl(){
            this.isDownloading = true;

            if(this.options.videoQuality === "Quality")
                this.options.videoQuality = "";

            if(!this.options.audioOnly && !this.options.playlist)
                this.options.soundQuality = this.getBestAudio;

            axios.post(`/download`,this.options)
            .then(result => {
                this.$emit('clicked', result.data);
                this.isDownloading = false;
                this.info = null;
                this.canDownload = false;
                this.options.url = "";
            })
            .catch(error => {
                this.canDownload = true;
                this.isDownloading = false;
                this.$parent.$refs.notificationComp.open('Error','Could not download video info. Please try again later.');
                console.error(error);
            });
        },
        getInfo(){
            // this.$parent.$refs.notificationComp.open('testa','hoihoi');

            if(this.options.url === ""){
                this.canDownload = false;
                return;
            }
            else if(this.options.audioOnly){
                this.canDownload = true;
                return
            }

            try{
                let id = null;

                if(RegExp(/((https:\/\/www\.reddit.com))/).test(this.options.url)){
                    this.options.target = "Reddit";
                    alert('s');
                    return;
                }
                else if(RegExp(/v=([0-9a-zA-Z$-_.+!*'(),]+)/).test(this.options.url)){
                    this.options.target = "Youtube";
                    this.isFetchingInfo = true;
                    const reg = this.options.url.match(/v=([0-9a-zA-Z$-_.+!*'(),]+)/);
                    id = reg[1];
                }

                if(this.isPlaylist(this.options.url)){
                    this.options.playlist = true;
                    this.canDownload = false;
                    this.options.url = "";
                    this.$parent.$refs.notificationComp.open('Information','Playlists downloads are not available. (yet)');
                    return;
                }

                axios.get(`/info/${id}`)
                .then(result => result.data)
                .then(result => {
                    this.info = result;
                    this.isFetchingInfo = false;
                    this.canDownload = true;
                })
                .catch(error => {
                    console.error(error);
                    this.canDownload = false;
                    this.isFetchingInfo = false;
                    this.$parent.$refs.notificationComp.open('Error','Could not fetch video info. Check your url and try again.');
                });
            }
            catch(error){
                this.isFetchingInfo = false;
                this.$parent.$refs.notificationComp.open('Error','Something went wrong. Please try again later.');
                console.error(error);
            }
        },
        selectAudioOnly(){
            if(this.options.audioOnly)
                this.options.audioOnly = false;
            else
                this.options.audioOnly = true;
        },
        isPlaylist(url){
            return RegExp(/list=([0-9a-zA-Z$-_.+!*'(),]+)/).test(url);
        }
    },
    computed: {
        getBestAudio(){
            let bestAudioID = null;

            for(let i = 0; i < this.info.formats.length; i++){
                const format = this.info.formats[i];

                if(format.ext == "m4a"){
                    bestAudioID = format.format_id;
                }
            }

            return bestAudioID;
        },
        getFormats(){
            const formats = [];

            for(let i = 0; i < this.info.formats.length; i++){
                const format = this.info.formats[i];

                if(format.ext == "mp4"){

                    if(formats.length === 0){
                        formats.push({
                            formatNote: format.format_note,
                            tbr: format.tbr,
                            formatId: format.format_id
                        });
                    }

                    let found = false;

                    for(let x = 0; x < formats.length; x++){
                        if(format.format_note === formats[x].formatNote){
                            found = true;

                            if(format.tbr > formats[x].tbr){
                                formats.splice(x, 1, {
                                    formatNote: format.format_note,
                                    tbr: format.tbr,
                                    formatId: format.format_id
                                });
                            }
                            break;
                        }
                    }

                    if(!found){
                        formats.push({
                            formatNote: format.format_note,
                            tbr: format.tbr,
                            formatId: format.format_id
                        });
                    }
                }
            }

            console.log(this.bestAudioID);

            return formats;
        }
    }
}
</script>
<style scoped>
input[type="url"]
{
    padding: 11px 5px;
    width: 411px;
    outline: none;
    border: none;
}
select
{
    padding: 11px;
    -webkit-appearance: button;
    border-radius: 0px;
    border: none;
    height: 46px;
    border-top-right-radius: 10px;
    border-bottom-right-radius: 10px;
    outline: none;
}
select:disabled
{
    border-color: rgba(118, 118, 118, 0.3);
    color: -internal-light-dark(graytext, rgb(170, 170, 170));
    background-color: rgb(194, 194, 194);
}
button
{
    padding: 11px;
    border: none;
    margin-left: -1px;
    outline: none;
    min-width: 127px;
}
.audio-button
{
    min-width: 70px;
    user-select: none;
    background: #FFF;
}
#audio-button-active
{
    -webkit-box-shadow: inset -1px 2px 24px 2px rgba(0,0,0,0.2);
    -moz-box-shadow: inset -1px 2px 24px 2px rgba(0,0,0,0.2);
    box-shadow: inset -1px 2px 5px 2px rgba(0,0,0,0.2);
    background: #2ecc71;
    color: white;
}
#audio-disabled-button
{
    border-color: rgba(118, 118, 118, 0.3);
    color: -internal-light-dark(graytext, rgb(170, 170, 170));
    background-color: -internal-light-dark(rgb(255, 255, 255), rgb(59, 59, 59));
}
</style>