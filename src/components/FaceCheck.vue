<template>
    <div>
        <div id="container" style="position: relative">
            <video id="camera" autoplay :style="cameraStyle"></video>
        </div>
        <h4 id="log" :style="labelStyle"></h4>
        <img :src="image" id="face" hidden>
    </div>
</template>

<script>
    // import nodejs bindings to native tensorflow,
    // not required, but will speed up things drastically (python required)
    import '@tensorflow/tfjs';

    // implements nodejs wrappers for HTMLCanvasElement, HTMLImageElement, ImageData

    import * as faceapi from 'face-api.js';

    export default {
        name: "FaceCheck",
        props: {
            image: {
                type: String,
                require: true
            },
            limitTime: {
                type: [Number, String],
                default: 1000
            },
            passRate: {
                type: Number,
                default: 0.4
            },
            delay: {
                type: Number,
                default: 100
            },
            modelDelay: {
                type: Number,
                default: 300
            },
            cameraStyle: {
                type: String,
                default: 'max-width: 50%;height: auto;border-radius: 5px;border: skyblue double 5px;'
            },
            labelStyle: {
                type: String,
                default: ''
            }
        },
        methods: {
            async initFaceApi() {
                const camera = document.getElementById('camera');
                camera.hidden = true;
                const log = document.getElementById('log');
                log.innerText = '加载模型中，请稍后';
                const MODEL_URL = '/models';
                Promise.all([
                    faceapi.nets.faceRecognitionNet.loadFromUri(MODEL_URL),
                    faceapi.nets.faceLandmark68Net.loadFromUri(MODEL_URL),
                    faceapi.nets.ssdMobilenetv1.loadFromUri(MODEL_URL),
                    faceapi.nets.faceRecognitionNet.loadFromUri(MODEL_URL),
                    faceapi.nets.faceExpressionNet.loadFromUri(MODEL_URL)
                ]).then(this.getFace)
            },
            initCamera(faceDetections) {
                const log = document.getElementById('log');
                log.innerText = '正在启动摄像头';
                const camera = document.getElementById('camera');
                navigator.getUserMedia({video: {}}, stream => camera.srcObject = stream, error => {
                    throw 'can not load media' + error
                });
                camera.addEventListener('play', () => {
                    log.innerText = '摄像头已启动';
                    camera.hidden = false;
                    const container = document.getElementById('container');
                    const canvas = faceapi.createCanvasFromMedia(camera);
                    canvas.getContext('2d').clearRect(0, 0, canvas.width, canvas.height);
                    const displaySize = {width: camera.offsetWidth, height: camera.offsetHeight};
                    faceapi.matchDimensions(canvas, displaySize);
                    container.append(canvas);
                    log.innerText = '正在识别。。。';
                    let that = this;
                    this.timer = setInterval(async () => {
                        const detections = await faceapi.detectAllFaces('camera').withFaceLandmarks().withFaceDescriptors().withFaceExpressions();
                        const resizeDetections = faceapi.resizeResults(detections, displaySize);
                        canvas.getContext('2d').clearRect(0, 0, canvas.width, canvas.height);
                        faceapi.draw.drawDetections(canvas, resizeDetections);
                        faceapi.draw.drawFaceExpressions(canvas, resizeDetections);
                        // console.info(resizeDetections)
                        if (detections.length === 1) {

                            if (faceDetections !== undefined && detections !== undefined && faceDetections.length === 1) {
                                let distance = await faceapi.euclideanDistance(resizeDetections[0].descriptor, faceDetections[0].descriptor);

                                log.innerText = '欧几里得度量：' + Math.round((distance) * 100) / 100;
                                faceapi.draw.drawFaceLandmarks(canvas, resizeDetections);
                                that.counter++;
                                if (distance < that.passRate) {
                                    log.innerText = '验证成功';
                                    canvas.getContext('2d').clearRect(0, 0, canvas.width, canvas.height);
                                    this.onCheckSuccess(distance);
                                    clearInterval(that.timer);
                                }
                                if (that.counter > that.limitTime) {
                                    log.innerText = '超过比对次数上限，验证失败';
                                    canvas.getContext('2d').clearRect(0, 0, canvas.width, canvas.height);
                                    that.onCheckFailed(distance);
                                    clearInterval(that.timer);
                                }
                            }
                        } else if (detections.length < 1) {
                            log.innerText = '未检测到人脸'
                        } else {
                            log.innerText = '多个人脸出现在摄像头中！'
                        }
                    }, this.delay)
                })
            },
            async getFace() {
                const log = document.getElementById('log');
                log.innerText = '模型已加载，正在加载人脸模板';
                let that = this;
                setTimeout(async () => {
                    var xhr = new XMLHttpRequest();
                    xhr.open('get', this.image, true);
                    xhr.responseType = 'blob';
                    xhr.onload = async function () {
                        if (this.status === 200) {
                            var blob = this.response;
                            await faceapi.bufferToImage(blob);
                            const faceDetections = await faceapi.detectAllFaces('face').withFaceLandmarks().withFaceDescriptors();
                            that.initCamera(faceDetections);
                        }
                    };
                    xhr.onerror = function () {
                        throw 'can not load face';
                    };
                    xhr.send()

                }, this.modelDelay)

            },
            onCheckSuccess(distance) {
                if (!this.success) {
                    this.success = true;
                    this.$emit('onCheckSuccess', distance)
                }

            },
            onCheckFailed(distance) {
                if (!this.failed) {
                    this.failed = true;
                    this.$emit('onCheckFailed', distance)
                }
            }
        },
        data() {
            return {
                counter: 0,
                timer: undefined,
                failed: false,
                success: false,
            }
        },
        mounted() {
            this.initFaceApi()
        }
    }
</script>

<style scoped>
    video {
        position: absolute;
        z-index: -1;
    }

    canvas {
        position: absolute;
    }
</style>
