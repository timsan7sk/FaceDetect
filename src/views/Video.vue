<script>
import * as faceAPI from 'face-api.js';
import { onMounted, reactive, ref } from 'vue';
export default {
    name: 'Video',
    setup() {
        const videoEl = ref(null);
        const canvasEl = ref(null);
        const visible = ref(false);
        const record = ref(false);
        const photo = ref(false);
        const videoRecord = ref(HTMLVideoElement);
        const snapShot = ref('');
        const getSnapshot = async () => {
            const video = document.getElementById('camera');
            let canvas = document.createElement('canvas');
            canvas.width = 640;
            canvas.height = 360;
            let ctx = canvas.getContext('2d');
            //draw image to canvas. scale to target dimensions
            ctx.drawImage(video, 0, 0, canvas.width, canvas.height);

            //convert to desired file format
            snapShot.value = canvas.toDataURL('image/png'); // can also use 'image/jpeg'
            photo.value = true;
        };
        const initParams = reactive({
            modelUri: '/models',
            option: new faceAPI.SsdMobilenetv1Options({ minConfidence: 0.98 })
        });
        const constraints = reactive({
            video: {
                width: {
                    min: 320,
                    ideal: 1280,
                    max: 1920
                },
                height: {
                    min: 240,
                    ideal: 720,
                    max: 1080
                },
                frameRate: {
                    min: 15,
                    ideal: 30,
                    max: 60
                },
                facingMode: 'user'
            }
        });
        /**
         * @function
         * @description toggle visibility
         */
        const expandEllipse = () => {
            // This is the canvas where you want to draw
            var canvas = document.getElementById('ellipse');
            var ctx = canvas.getContext('2d');
            for (let i = 1; i <= 20; i++) {
                ctx.reset();
                ctx.beginPath();
                ctx.ellipse(320, 180, 100 + i, 150 + i, 0, 0, 2 * Math.PI);
                ctx.lineWidth = 5;
                ctx.strokeStyle = 'red';
                ctx.stroke();
                // ctx.arc(320, 180, 100, 0, 2 * Math.PI);
                ctx.rect(640, 0, -640, 640);
                // ctx.restore();
                ctx.fillStyle = 'white';
                ctx.fill();
            }
        };
        const getRecord = async () => {
            const stream = await navigator.mediaDevices.getUserMedia(constraints);
            const mediaRecorder = new MediaRecorder(stream, {
                mimeType: 'video/webm'
            });
            mediaRecorder.start();
            getTimer();
            setTimeout(() => mediaRecorder.stop(), 5000);
            mediaRecorder.ondataavailable = (event) => {
                videoRecord.value.src = URL.createObjectURL(event.data);
            };
            setTimeout(() => (record.value = true), 5000);
        };
        const getTimer = async () => {
            let timeleft = 5;
            let getInteval = setInterval(() => {
                if (timeleft > 0) {
                    document.getElementById('countdown').innerHTML = timeleft;
                } else {
                    clearInterval(getInteval);
                    document.getElementById('countdown').innerHTML = '';
                }
                --timeleft;
            }, 1000);
        };
        /**
         * @function
         * @description detect input video
         */
        const runModel = async () => {
            const result = await faceAPI.detectAllFaces(videoEl.value, initParams.option).withFaceLandmarks(true).withFaceDescriptors(true);
            if (result) {
                const dims = faceAPI.matchDimensions(canvasEl.value, videoEl.value, true);
                const resizeResults = faceAPI.resizeResults(result, dims);
                try {
                    const points = resizeResults[0].landmarks.getJawOutline();

                    let tmp_xy = 0.0;
                    let k = 0;
                    for (let i = 0; i < 17; i++) {
                        // console.log(points[i]);
                        tmp_xy = ((points[i].x - 640) / 200.0) ** 2 + ((points[i].y - 360) / 300.0) ** 2;
                        if (tmp_xy <= 0.9 && tmp_xy > 0.75) {
                            k++;
                        }
                    }
                    k > 8 ? (visible.value = true) : (visible.value = false);
                } catch (e) {
                    console.log(e);
                }
            }
            setTimeout(() => runModel());
        };

        onMounted(() => {
            /**
             * @function
             * @description create ellipse in canvas and reverse fill
             */
            const doEllipse = async () => {
                // This is the canvas where you want to draw
                var canvas = document.getElementById('ellipse');
                var ctx = canvas.getContext('2d');
                // ctx.save();
                ctx.beginPath();
                ctx.ellipse(320, 180, 100, 150, 0, 0, 2 * Math.PI);
                ctx.lineWidth = 5;
                ctx.strokeStyle = 'red';
                ctx.stroke();
                // ctx.arc(320, 180, 100, 0, 2 * Math.PI);
                ctx.rect(640, 0, -640, 640);
                // ctx.restore();
                ctx.fillStyle = 'white';
                ctx.fill();
            };
            /**
             * @function
             * @description load the trained model
             */
            const initModel = async () => {
                await faceAPI.nets.faceLandmark68TinyNet.loadFromUri(initParams.modelUri);
                await faceAPI.nets.ssdMobilenetv1.loadFromUri(initParams.modelUri);
                await faceAPI.nets.faceLandmark68Net.loadFromUri(initParams.modelUri);
                await faceAPI.nets.faceRecognitionNet.loadFromUri(initParams.modelUri);
            };

            /**
             * @function
             * @description startup webcam
             */
            const startStream = async () => {
                try {
                    const stream = await navigator.mediaDevices.getUserMedia(constraints);
                    videoEl.value.srcObject = stream;
                } catch (error) {
                    console.error(error.message);
                }
            };
            doEllipse();
            initModel();
            startStream();
        });

        return {
            videoEl,
            canvasEl,
            runModel,
            getSnapshot,
            getRecord,
            expandEllipse,
            getTimer,
            videoRecord,
            snapShot,
            visible,
            photo,
            record
        };
    }
};
</script>

<template>
    <div class="flex flex-wrap align-items-center justify-content-center card-container blue-container">
        <div class="card shadow-1 py-8 px-8" style="border-radius: 50px">
            <div class="flex justify-content-center flex-wrap card-container">
                <div class="flex align-items-center justify-content-center border-round">
                    <!-- <ProgressSpinner class="loader" aria-label="Loading" /> -->
                    <video id="camera" ref="videoEl" autoplay="true" playsinline @loadedmetadata="runModel" />
                    <canvas class="border" ref="canvasEl" />
                    <canvas width="640" height="360" id="ellipse"></canvas>

                    <div id="countdown" class="absolute text-7xl"></div>

                    <div class="glow" v-if="visible"></div>
                </div>
            </div>
            <div class="flex justify-content-center flex-wrap card-container">
                <div class="flex align-items-center justify-content-center text-gray-900 border-round m-2">
                    <Button v-tooltip.bottom="'Get Photo'" @click="getSnapshot" label="" icon="pi pi-camera" class="z-5"></Button>
                </div>
                <div class="flex align-items-center justify-content-center text-gray-900 border-round m-2">
                    <Button v-tooltip.bottom="'Get Video'" @click="getRecord" label="" icon="pi pi-video" class="z-5"></Button>
                </div>
                <div class="flex align-items-center justify-content-center m-2"></div>
            </div>
        </div>
    </div>
    <Dialog header="Video" v-model:visible="record" :breakpoints="{ '960px': '75vw' }" :style="{ width: '40vw' }" :modal="true">
        <div class="flex align-items-center justify-content-center">
            <video controls playsinline ref="videoRecord"></video>
        </div>
    </Dialog>
    <Dialog header="Photo" v-model:visible="photo" :breakpoints="{ '960px': '75vw' }" :style="{ width: '40vw' }" :modal="true">
        <div class="flex align-items-center justify-content-center">
            <Image :src="snapShot" alt="Photo" />
        </div>
    </Dialog>
</template>

<style lang="scss" scoped>
video {
    width: 640px;
    height: 360px;
}
#camera {
    position: relative;
    object-fit: contain;
    width: 640px;
    height: 360px;
    z-index: 0;
    margin: 0;
    padding: 0;
    -o-transform: scaleX(-1);
    -moz-transform: scaleX(-1);
    -webkit-transform: scaleX(-1);
    -ms-transform: scaleX(-1);
    transform: scaleX(-1);
}
#ellipse {
    position: absolute;
    z-index: 1;
    opacity: 1;
}
.border {
    position: absolute;
    z-index: 3;
    width: 640px;
    height: 360px;
}
.glow {
    position: absolute;
    z-index: 5;
    width: 202px;
    height: 302px;
    border-radius: 50%;
    border: 5px solid #aef4af;
    animation: glow 1s infinite alternate;
}

@keyframes glow {
    from {
        box-shadow: 0 0 10px -10px #aef4af;
    }
    to {
        box-shadow: 0 0 10px 15px #aef4af;
    }
}
</style>
