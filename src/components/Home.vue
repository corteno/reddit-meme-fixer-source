<template>
    <div class="home-wrapper">
        <h1 class="page-title">Reddit Title Fixer</h1>
        <div class="page-content">
            <h2 class="page-subtitle">
                Don't you hate when the punchline of a meme is in the title and can't link it to your friends?
                Fear no more, just paste the link and you'll get your meme back!
            </h2>
            <div class="form-wrapper">
                <form @submit="onSubmit" class="form">
                    <input type="text" :class="this.value !== '' ? 'text-input active-text-input' : 'text-input'"
                           v-model="inputValue">
                    <label for="title-lightmode" class="row" title="Switches the title background to white and the text to black">
                        <input type="checkbox" id="title-lightmode" class="checkbox" v-model="lightMode">
                        <p>Title light mode</p>
                    </label>

                    <input type="submit" v-if="!isLoading" value="Submit" :class="'submit-button'"
                           :disabled="this.value === ''">
                    <div v-if="isLoading" class="lds-dual-ring"></div>
                </form>
                <p class="error-message">{{this.errorMessage}}</p>

            </div>


            <Canvas ref="canvas" width="600" height="600"></Canvas>

            <img src="" alt="" ref="img" class="meme-img">
        </div>
        <footer>
            <p>
                Made by <a href="http://borsodidavid.com">Dávid Borsodi</a> © {{new Date().getFullYear()}}
            </p>
        </footer>
    </div>
</template>

<script>
    import psl from 'psl';
    import axios from 'axios';

    export default {
        name: "Home",
        computed: {
            inputValue: {
                get() {
                    return this.value;
                },
                set(val) {
                    this.value = val;
                }
            }
        },
        data: function () {
            return {
                value: '',
                errorMessage: '',
                textBlockHeight: 33,
                characterLimitPerLine: '55',
                isLoading: false,
                lightMode: false
            }

        },
        methods: {
            onSubmit: function (e) {
                e.preventDefault();
                if (this.validateUrl(this.value)) {
                    let url = this.value + ".json";

                    axios.post('https://reddit-meme-fixer.herokuapp.com/meme', {
                        data: url
                    }).then((response) => {
                        console.log(response.data.title.length);
                        this.displayOnCanvas(response.data);
                        //this.$refs.img.src = response.data.img;
                    });

                    this.isLoading = true;
                }
            },
            displayOnCanvas(data) {
                let canvas = this.$refs.canvas;
                let context = canvas.getContext('2d');
                let imageObj = new Image();
                imageObj.src = data.img;

                imageObj.onload = () => {
                    //let ratio = imageObj.width / imageObj.height;

                    context.canvas.width = imageObj.width;


                    //context.textAlign = 'center';

                    if (data.title.length > this.characterLimitPerLine) {
                        let newTitle = this.insertLineBreak(data.title);

                        context.canvas.height = imageObj.height + (this.textBlockHeight * newTitle.length);
                        context.drawImage(imageObj, 0, (this.textBlockHeight * newTitle.length));

                        context.fillStyle = !this.lightMode ? 'black' : 'white';
                        context.fillRect(0, 0, imageObj.width, (this.textBlockHeight * newTitle.length));


                        context.font = "26px Arial";
                        context.fillStyle = this.lightMode ? 'black' : 'white';

                        for (let i = 0; i < newTitle.length; i++) {
                            context.fillText(newTitle[i], 10, 30 * (i + 1));
                        }
                    } else {
                        context.canvas.height = imageObj.height + (this.textBlockHeight * 2);
                        context.drawImage(imageObj, 0, (this.textBlockHeight * 2));

                        context.fillStyle = !this.lightMode ? 'black' : 'white';
                        context.fillRect(0, 0, imageObj.width, (this.textBlockHeight * 2));

                        context.font = "26px Arial";
                        context.fillStyle = this.lightMode ? 'black' : 'white';
                        context.fillText(data.title, 10, 45);
                    }


                    this.isLoading = false;
                    this.value = '';
                    this.$refs.img.src = canvas.toDataURL();

                };

                imageObj.src = data.img;


            },
            getSpaces(string) {
                let spaces = [];

                for (let i = 0; i < string.length; i++) {
                    //console.log(string[i]);
                    if (string[i] === ' ') {
                        spaces.push(i);
                    }
                }

                return spaces;
            },
            insertLineBreak(string) {
                let lines = Math.ceil(string.length / this.characterLimitPerLine);
                let currentString = string;
                let returnString = [];

                //console.log(currentString);


                for (let i = 0; i < lines; i++) {
                    let spaces = this.getSpaces(currentString);
                    console.log("spaces:", spaces);

                    if(spaces.length > 0) {
                        let closest = spaces.reduce((prev, curr) => {
                            return (Math.abs(curr - this.characterLimitPerLine) < Math.abs(prev - this.characterLimitPerLine) ? curr : prev);
                        });

                        if (currentString.length >= this.characterLimitPerLine) {
                            let newLine = currentString.slice(0, closest + 1);
                            newLine = newLine.trim();
                            returnString.push(newLine); //Adding the sliced string to the return array


                            currentString = currentString.substring(closest , currentString.length);
                        } else {
                            returnString.push(currentString.trim());
                        }
                    } else {
                        returnString.push(currentString);
                    }

                }

                return returnString;
            },
            setErrorMessage(message) {
                this.errorMessage = message;
                setTimeout(() => {
                    this.errorMessage = '';
                }, 5000);
            },
            validateUrl(str) {
                let pattern = new RegExp('^(https?:\\/\\/)?' + // protocol
                    '((([a-z\\d]([a-z\\d-]*[a-z\\d])*)\\.)+[a-z]{2,}|' + // domain name
                    '((\\d{1,3}\\.){3}\\d{1,3}))' + // OR ip (v4) address
                    '(\\:\\d+)?(\\/[-a-z\\d%_.~+]*)*' + // port and path
                    '(\\?[;&a-z\\d%_.~+=-]*)?' + // query string
                    '(\\#[-a-z\\d_]*)?$', 'i'); // fragment locator

                let isUrl = !!pattern.test(str);

                if (isUrl) {
                    let domainName = psl.get(this.extractHostName(str));
                    if (domainName === 'reddit.com' || domainName === 'old.reddit.com') {
                        return true;
                    }
                }
                this.setErrorMessage('Invalid URL, can only fetch memes from reddit.com');
                return false
            },
            extractHostName(url) {
                let hostname;
                //find & remove protocol (http, ftp, etc.) and get hostname

                if (url.indexOf("//") > -1) {
                    hostname = url.split('/')[2];
                } else {
                    hostname = url.split('/')[0];
                }

                //find & remove port number
                hostname = hostname.split(':')[0];
                //find & remove "?"
                hostname = hostname.split('?')[0];

                return hostname;
            }

        }
    }
</script>

<style>

    html {
        overflow-x: hidden;
    }

    body {
        margin: 0;
    }

    * {
        font-family: UbuntuRegular, sans-serif;
    }

    #app {
        display: flex;
        justify-content: center;
        margin-top: 0 !important;
    }

    label {
        display: flex;
        justify-content: center;
        margin-top: 20px;
    }

    p {
        margin: 0;
    }

    label p {
        display: flex;
        align-self: center;
        font-family: UbuntuLight, sans-serif;
    }

    .checkbox {
        position: relative;
        appearance: none;
        height: 20px;
        width: 40px;
        margin-right: 10px;

        border-radius: 20px;

        border: 1px solid #ccc;
        cursor: pointer;

        transition: 0.2s border-color;
    }

    .checkbox:after {
        position: absolute;
        content: '';
        height: 16px;
        width: 16px;
        left: 1px;
        top: 1px;
        border-radius: 20px;
        background: #ccc;

        transition: 0.2s left, 0.2s background-color;
    }

    .checkbox:checked {
        border-color: #2c3e50;
    }

    .checkbox:checked:after {
        left: 20px;
        background-color: #2c3e50;
    }


    .meme-img {
        max-width: 700px;
        width: 100%;
    }

    .page-title {
        position: fixed;
        text-transform: uppercase;
        font-family: UbuntuLight, sans-serif;
        font-weight: 300;
        width: 100vw;
        top: 0;
        left: 0;
        background: #FF4500;
        color: #f2f2f2;
        margin: 0;
        padding: 20px 0;
    }

    .page-subtitle {
        font-family: UbuntuLight, sans-serif;
        font-weight: 300;
        margin-top: 90px;
    }

    .page-content {
        padding: 0 20px;
    }

    .home-wrapper {
        display: flex;
        flex-direction: column;
        max-width: 800px;
        width: 100%;
        align-self: center;
    }

    canvas {
        position: absolute;
        right: 9000px;
    }

    .form-wrapper, .form-wrapper > * {
        display: flex;
    }

    .form-wrapper {
        width: 100%;
        flex-direction: column;
        align-items: center;
    }

    .form {
        width: 100%;
        flex-direction: column;
        align-items: center;
        margin-bottom: 30px;
    }

    .text-input {
        width: 100%;
        font-size: 16px;
        padding: 0 5px;
        height: 30px;
        border-radius: 6px;
        border: 2px solid #ccc;
        align-self: center;
        transition: 0.2s border-color;
    }

    .active-text-input {
        border-color: #2c3e50;
    }

    .text-input:focus {
        border-color: #2c3e50;
    }

    .submit-button {
        appearance: none;
        -webkit-appearance: none;
        -moz-appearance: none;
        background: transparent;
        border: 2px solid #9f9f9f;
        color: #9f9f9f;
        border-radius: 8px;
        padding: 10px 20px;
        margin-top: 30px;
        align-self: center;
        font-family: UbuntuBold, sans-serif;
        text-transform: uppercase;
        cursor: pointer;
        transition: 0.2s color, 0.2s border-color;
    }

    .submit-button:not(:disabled):hover {
        border-color: #2c3e50;
        color: #2c3e50;
    }


    .lds-dual-ring {
        display: inline-block;
        width: 64px;
        height: 64px;
        margin-top: 15px;
    }

    .lds-dual-ring:after {
        content: " ";
        display: block;
        width: 36px;
        height: 36px;
        margin: 8px;
        border-radius: 50%;
        border: 6px solid #2c3e50;
        border-color: #2c3e50 transparent #2c3e50 transparent;
        animation: lds-dual-ring 1.2s linear infinite;
    }

    @keyframes lds-dual-ring {
        0% {
            transform: rotate(0deg);
        }
        100% {
            transform: rotate(360deg);
        }
    }

    footer {
        display: flex;
        position: fixed;
        bottom: 0;
        left: 0;
        height: 40px;
        width: 100%;
        background: #FF4500;
        justify-content: center;
        align-items: center;

        color: #f2f2f2;
    }

    footer p {
        font-family: UbuntuLight, sans-serif;
        font-weight: 200;
    }

    footer a {
        display: inline;
        color: #f2f2f2;
    }

    @media screen and (max-width: 500px) {
        .page-subtitle {
            font-size: 20px;
        }

        .submit-button {
            margin-top: 15px;
        }
    }


    @font-face {
        font-family: UbuntuRegular;
        src: url("../assets/fonts/Ubuntu-Regular.ttf");
    }

    @font-face {
        font-family: UbuntuLight;
        src: url("../assets/fonts/Ubuntu-Light.ttf");
    }

    @font-face {
        font-family: UbuntuMedium;
        src: url("../assets/fonts/Ubuntu-Medium.ttf");
    }

    @font-face {
        font-family: UbuntuBold;
        src: url("../assets/fonts/Ubuntu-Bold.ttf");
    }

</style>