<script setup>
</script>

<template>
    <div class="container">
    <div class="info">
      현재 FACTORIOGIF는 다음의 사양을 가집니다.
      <ul>
        <li>40x40 ~ 80x80 픽셀 (사이즈는 자동 조절됨)</li>
        <li>최대 12 프레임</li>
      </ul>
    </div>

    <div class="controls">
      <div class="file-input">
        <input type="file" id="uploadImage" multiple required accept="image/*" @change="GetFileName($event.target.files)" onclick="this.value=null;"/>
        <label class="upload-btn" for="uploadImage">파일 선택</label>
      </div>
      <button @click="ClearImage()">초기화</button>
    </div>

    <div class="delay-input">
      <label for="delay">대기 시간 (초)</label>
      <input type="number" id="delay" :value="time" step="0.1" min="0.1" max="100" @input="ChangeSpeed($event.target.value)" />
    </div>

    <div class="delay-input">
      <label for="col">크기</label>
      <input type="number" id="col" :value="size" min="4" max="8" @input="ClearImage($event.target.value)" />
    </div>

    <div>
        <img class="preview" id="previewImage" :width="previewSize" :height="previewSize" :src="getImgUrl('black.jpg')"/>
    </div>
    <div>
        <div class="preview" id="loadedimgList"></div>
    </div>

    <button class="generate-btn" @click="MakeBlueprintString()">생성</button>

    <textarea placeholder="생성을 누르고 문자열을 팩토리오에 붙여넣기하세요." :value="blueprintString"></textarea>
  </div>
</template>

<script>
import { defineComponent } from 'vue';
import { parseGIF, decompressFrames } from 'gifuct-js';

export default defineComponent({
    data() {
        return {
            time : 1,
            split : 10,
            maxFrame : 12,
            size : 4,
            realSize : 40,
            previewSize : 200,
            canvasList : [],
            gifInterval : null,
            blueprintString : ""
        }
    },
    methods : {
        ReadFile(file) {
            return new Promise((resolve) => {
                let f = new FileReader();
                if (file.type == 'image/gif') {
                    f.readAsArrayBuffer(file);
                    f.onload = async (event) => {
                        const gifImg = parseGIF(event.target.result);
                        const frames = decompressFrames(gifImg, true);

                        const gifCanvas = document.createElement('canvas');
                        gifCanvas.width = frames[0].dims.width;
                        gifCanvas.height = frames[0].dims.height;
                        const gifCtx = gifCanvas.getContext('2d', { willReadFrequently: true });
                        const gifImageData = gifCtx.createImageData(gifCanvas.width, gifCanvas.height);

                        for (var i = 0; i < frames.length; i++) {
                        //for (var i = 1; i < 2; i++) {
                            await this.AddCanvasImage(gifCanvas, frames[i]);
                        }
                        resolve(true);
                    };
                } else {
                    f.readAsDataURL(file);
                    f.onload = async (event) => {
                        const img = new Image();
                        img.src = event.target.result;
                        img.onload = async () => {
                            await this.AddCanvasImage(img);
                            resolve(true);
                        };
                    };
                }
            })
        },
        async AddCanvasImage(img, frame) {
            if (!img) {
                return;
            }
            if (frame) {
                var dims = frame.dims;
                const tempCanvas = document.createElement('canvas');
                tempCanvas.width = dims.width;
                tempCanvas.height = dims.height;
                const tempCtx = tempCanvas.getContext('2d', { willReadFrequently: true });
                const tempImageData = tempCtx.createImageData(tempCanvas.width, tempCanvas.height);
                tempImageData.data.set(frame.patch);
                tempCtx.putImageData(tempImageData, 0, 0);

                const gifCtx = img.getContext('2d', { willReadFrequently: true });
                gifCtx.drawImage(tempCanvas, dims.left, dims.top);
            }

            const realCanvas = document.createElement('canvas');
            const ctx = realCanvas.getContext('2d', { willReadFrequently: true });

            const ratio = Math.min(img.width, img.height) / Math.max(img.width, img.height);
            var realSize = this.split * this.size;
            realCanvas.width = realSize;
            realCanvas.height = realSize;
            if (img.width > img.height) {
                ctx.drawImage(img, 0, 0, realSize, realSize * ratio);
            } else {
                ctx.drawImage(img, 0, 0, realSize * ratio, realSize);
            }

            const cutCanvas = document.createElement('canvas');
            const ctx2 = cutCanvas.getContext('2d', { willReadFrequently: true });
            const cutSize = realSize + (Math.floor(realSize / this.split) - 1) * 2;
            cutCanvas.width = cutSize;
            cutCanvas.height = cutSize;
            cutCanvas.className = "loadedImg";

            for (var i = 0; i < realSize; i++) {
                for (var j = 0; j < realSize; j++) {
                    const pixel = ctx.getImageData(i, j, 1, 1);
                    ctx2.putImageData(pixel, i + Math.floor(i / this.split) * 2, j + Math.floor(j / this.split) * 2);
                }
            }

            const previewImage = document.getElementById('previewImage');
            previewImage.src = cutCanvas.toDataURL();

            this.canvasList.push({
                real : realCanvas,
                cut : cutCanvas
            });
            const imglist = document.getElementById('loadedimgList');
            imglist.appendChild(realCanvas);
        },
        async GetFileName(files) {
            let allowedExtension = ['image/jpeg', 'image/jpg', 'image/png','image/gif','image/bmp'];
            if (files) {
                this.StopGIF();

                for (var i = 0; i < files.length; i++) {
                    if (allowedExtension.indexOf(files[i].type) > -1) {
                        await this.ReadFile(files[i]);
                    }
                }

                this.StartGIF();
            }
        },
        getUrl(img) {
            return `./assets/${img}`;
        },
        getImgUrl(img) {
            return this.getUrl(`images/` + img);
        },
        ClearImage(size) {
            if (size) {
                this.size = size;
            }
            this.StopGIF();

            const previewImage = document.getElementById('previewImage');
            previewImage.src = this.getImgUrl('black.jpg');

            this.canvasList = [];
            const imglist = document.getElementById('loadedimgList');
            while (imglist.firstChild) {
                imglist.firstChild.remove();
            }
        },
        StopGIF() {
            this.blueprintString = "";
            if (this.gifInterval) {
                clearInterval(this.gifInterval);
            }
        },
        StartGIF() {
            console.log(this.time * 1000);
            let count = this.canvasList.length;
            const previewImage = document.getElementById('previewImage');
            this.gifInterval = setInterval(() => {
                var length = this.canvasList.length;
                if (count > length || length == 0) {
                    this.StopGIF();
                    return;
                }
                else if (count == length) {
                    count = 0;
                }
                previewImage.src = this.canvasList[count].cut.toDataURL();
                count++;
            }, this.time * 1000);
        },
        ChangeSpeed(value) {
            this.time = value;
            this.StopGIF();
            this.StartGIF();
        },
        GetPixel(imageData, index) {
            var i = index * 4, d = imageData.data;
            return [d[i],d[i+1],d[i+2],d[i+3]];
        },
        GetPixelXY(imageData, x, y) {
            return this.GetPixel(imageData, y * imageData.width + x);
        },
        GetColorCount(pixel) {
            return pixel[0] * 65536 + pixel[1] * 256 + pixel[2];
        },
        IsNum(value) {
            const pattern = /^-?\d*(\.\d*)?$/;
            return pattern.test(value);
        },
        MakeBlueprintString() {
            var length = this.canvasList.length;
            if (length > this.maxFrame) {
                length = this.maxFrame;
            }
            const row = this.size;
            fetch(this.getUrl(`json/internal.json`))
            .then(resp => resp.blob())
            .then(blob => {
                let f = new FileReader();
                f.readAsText(blob);
                f.onload = async (event) => {
                    const defaultString = event.target.result;
                    const imageList = [];
                    for (var c = 0; c < length; c++) {
                        const internalList = [];
                        imageList.push(internalList);
                        var canvas = this.canvasList[c].real;
                        const ctx = canvas.getContext('2d', { willReadFrequently: true });
                        var data = ctx.getImageData(0, 0, canvas.width, canvas.height);
                        for (var i = 0; i < row; i++) {
                            for (var j = 0; j < row; j++) {
                                internalList[i + j * row] = JSON.parse(defaultString);
                                for (var x = 0; x < this.split; x++) {
                                    for (var y = 0; y < this.split; y++) {
                                        var color = this.GetColorCount(this.GetPixelXY(data, x + i * this.split, y + j * this.split));
                                        internalList[i + j * row].filters[x + y * this.split].count = color;
                                    }
                                }
                            }
                        }
                    }
                    fetch(this.getUrl(`json/base` + row + `.json`))
                    .then(resp => resp.blob())
                    .then(blob => {
                        let f = new FileReader();
                        f.readAsText(blob);
                        f.onload = async (event) => {
                            const baseJson = JSON.parse(event.target.result);
                            const entities = baseJson.blueprint.entities;
                            const baseList = [];
                            for (var i = 0; i < entities.length; i++) {
                                if (entities[i].name == "constant-combinator") {
                                    var section = entities[i].control_behavior.sections.sections;
                                    if (section.length == 1) {
                                        var temp = section[0].filters;
                                        if (temp.length == 4 && temp[0].name == "signal-check" && temp[1].name == "signal-dot" && temp[2].name == "signal-info" && temp[3].name == "signal-deny") {
                                            var time = 1.0;
                                            if (this.IsNum(this.time)) {
                                                time = this.time;
                                            }
                                            if (time < 0.1) {
                                                time = 0.1;
                                            }
                                            temp[2].count = Math.floor(time * 60);
                                            temp[3].count = temp[2].count * imageList.length - 1;
                                        }
                                        else if (temp.length == 3 && temp[0].name == "signal-dot" && temp[1].name == "signal-X" && temp[2].name == "signal-Y") {
                                            const num = (temp[2].count - 1) + (temp[1].count - 1) * row;
                                            if (!baseList[num]) {
                                                baseList[num] = [];
                                            }
                                            baseList[num].push(section);
                                            temp[0].count = baseList[num].length;
                                        }
                                    }
                                }
                            }
                            for (var c = 0; c < imageList.length; c++) {
                                var internalList = imageList[c];
                                for (var i = 0; i < row; i++) {
                                    for (var j = 0; j < row; j++) {
                                        var internal = internalList[i + j * row];
                                        if (internal) {
                                            baseList[i + j * row][c].push(internal);
                                        }
                                    }
                                }
                            }
                            const compressedData = pako.deflate(JSON.stringify(baseJson), { level: 9 });
                            const bin = [];
                            for (let i = 0; i < compressedData.length; i++) {
                                bin.push( String.fromCharCode( compressedData[ i ] ) );
                            }
                            const as_text = btoa( bin.join( "" ) );
                            this.blueprintString = "0" + as_text;
                        };
                    });
                };
            });
        }
    },
})
</script>
<style scoped>
  .container {
    background-color: #2a2a40;
    padding: 32px;
    border-radius: 12px;
    box-shadow: 0 4px 20px rgba(0, 0, 0, 0.4);
    width: 400px;
  }

  .info {
    font-size: 14px;
    margin-bottom: 20px;
    line-height: 1.6;
  }

  .info ul {
    margin-top: 8px;
    padding-left: 20px;
  }

  .controls {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 16px;
  }

  input[type="file"] {
    display: none;
  }

  label.upload-btn {
    display: inline-block;
    padding: 8px 14px;
    background-color: #3d7eff;
    color: white;
    border-radius: 6px;
    cursor: pointer;
  }

  button {
    padding: 8px 14px;
    background-color: #ff5c5c;
    color: white;
    border: none;
    border-radius: 6px;
    cursor: pointer;
  }

  .delay-input {
    display: flex;
    align-items: center;
    justify-content: space-between;
    margin-bottom: 16px;
  }

  .delay-input label {
    flex: 1;
  }

  .delay-input input[type="number"] {
    width: 80px;
    padding: 6px;
    border: 1px solid #555;
    border-radius: 6px;
    background-color: #1e1e2f;
    color: #e0e0e0;
  }

  .preview {
    display: block;
    background-color: black;
    margin: 0 auto 20px;
    border: 2px solid limegreen;
    border-radius: 6px;
  }

  .generate-btn {
    display: block;
    width: 100%;
    padding: 10px;
    background-color: #00c88a;
    color: white;
    font-weight: bold;
    border: none;
    border-radius: 6px;
    cursor: pointer;
    margin-bottom: 16px;
  }

  textarea {
    width: 100%;
    height: 60px;
    border: 1px solid #555;
    border-radius: 6px;
    padding: 8px;
    background-color: #1e1e2f;
    color: #e0e0e0;
    resize: none;
  }
</style>