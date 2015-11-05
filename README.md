WebSocket/Web Audio API for Ispikit - version 1.1
=================================================

This document describes the API and usage of the client-side module of the client-server version of the Ispikit pronunciation technology. Making use of WebSockets, the Web Audio API and WebRTC, pronunciation assessment applications can run directly in the web browser without requiring any browser plug-in. The content of this SDK is:

* `audioRecorder.js`: JavaScript implementation of audio recording and interface with WebSocket server.
* `audioRecorderWorker.js`: Web Worker code used by the AudioRecorder.
* `wsispikit.html`: Sample web application that demonstrates the use of the API.

Content:

1. Trying the sample application
2. Introduction
3. SDK usage
4. Limitations and known issues


# 1. Trying the sample application

* Copy the 3 files (`audioRecorder.js`, `audioRecorderWorker.js`, `wsispikit.html`) into a location served by a web server and open wsispikit.html with Google Chrome or Firefox. Web audio does not work if the page is served from "file://...".
* Enter a sentence in the input field.
* Press `Start` and read the sentence.
* While you read, the audio volume is updated.
* Press `stop`.
* You should see the result or possibly an error message.

# 2. Introduction

The Ispikit WebSocket server exposes a simple WebSocket API to receive and analyze audio data. It is intended to be used together with the Web Audio API and WebRTC making it a complete SDK for web-only pronunciation assessment. Audio data can alternatively come from other sources, but it is not documented here.

# 3. SDK usage

Let's start with an outline of what an application needs to do:

* Create a WebSocket connection to the Ispikit server.
* Listen to the messages from the WebSocket. These messages can be either the result of analysis (score) or error codes.
* Initialize an instance of AudioRecorder (let's call it `recorder`), defined in `audioRecorder.js`. It exposes a simple API for recording audio and communicates directly with the WebSocket.
* React to callbacks from the `recorder`. The `recorder` calls back with audio volume information in real time and with a blob containing the last audio file that has been recorded, at the end of each recording.

These steps are all implemented in the sample application, we describe them here in more details.

## 3.a WebSocket

A WebSocket object must be created with the URL of the Ispikit server (currently for the Ispikit server "ws://108.59.3.115:18875"). The application should assign a callback to `onopen` and `onmessage`. 

* `onopen` is called once the connection is established. The app should not try to start recording audio before the WebSocket is connected.
* `onmessage` is used by the WebSocket to send messages back to the client. They are encoded in JSON format, so a message must first be parsed:

    ```javascript
    websocket.onmessage = function(e) {
    var message = JSON.parse(e.data);
    ...
    ```

    The message contains either the result of analysis or an error code.


Following is the content of these messages, after they are parsed (converted from a JSON string to a JavaScript object):

* `message` might contain an analysis result. In this case, it contains 3 fields: `score`, `speed` and `words`, such as `{score: 85, speed: 73, words: "..."}`. `score` is the overall assessment score, between 0 (lowest score) and 100 (highest pronunciation score). `speed` is the measure of speech tempo, measure in phonemes in 10 seconds. A value lower than 80 could be considered slow or not fluent. `words` indicate which words have been recognized and mispronounced, the format is as follows: `words` is a string which includes the recognized words, separated by commas, in the form i-j-k-l where:
    * i is the index of the sentence being practiced.
    * j is the index of the word within the sentence, starting from 0.
    * k and l give information on the pronunciation alternative the user said, which is not useful/usable for now.

    In addition, any word can be followed by a pattern such as MISP-i which means that the ith phoneme was mispronounced. That mispronounced phoneme detection should be used carefully as it is not guaranteed to be accurate. However, it can be implied that a word with at least one mispronounced phoneme is a mispronounced word.

    As an example, say you have the sentence "He speaks English and he likes it", and you get:

    ```javascript
    result.words="1-0-0-0, 1-1-2-1, 1-2-0-0 MISP-1 MISP-2, 1-4-0-0, 1-5-2-0 MISP-2"
    ```

    It means that the learner did not say the words "and" and "it", and mispronounced the words "English" and "likes".
* `message` might contain an error code. In this case, it looks like `{error: "ERROR_CODE"}`, where ERROR_CODE can be either:
    * `NO_AUDIO`: No audio was received by the server, probably because the time between calls to start and stop was shorter than a buffer length.
    * `INVALID_SENTENCE`: The provided sentence is not valid, for instance it contains words that are not in the dictionary. We'll see below how the sentence is given.

## 3.b AudioRecorder

The file `audioRecorder.js` defines an `AudioRecorder` class that deals with audio recording and sending data to the server through the WebSocket. It is designed to be generic but you might choose to adjust it, making sure the interaction with the WebSocket server is not affected. It was created based on the Recoderjs project (https://github.com/mattdiamond/Recorderjs).

* Constructor: The constructor takes 4 arguments, in order: `(source, websocket, config, callback)`.
    * `source` is the Media Stream Source created by createMediaStreamSource in the callback of `getUserMedia`. For instance, if getusermedia is called with:

        ```javascript
        navigator.getUserMedia({audio: true}, startUserMedia, function(e) { /* Error callback */ });
        ```

        then we might have:

        ```javascript    
        function startUserMedia(stream) {
          var input = audio_context.createMediaStreamSource(stream);
          input.connect(audio_context.destination);
          recorder = new AudioRecorder(input, websocket, {}, callback);
        }
        ```
    *  `websocket`: The websocket object. It can also be set later as it is a public member. For instance, in `onopen` on WebSocket, you might want to do:

        ```javascript
           recorder.websocket = websocket;
        ```
    * `config`: A config object. Currently, the only thing you might want to include is a path to the web worker file: `{worker: "/path/to/audioRecorderWorker.js"}`. Otherwise it is assumed to be a global variable `AUDIO_RECORDER_WORKER`. Make sure at least one of the two is set.
    * `callback`: callback function that takes one argument and that will be called by the recorder to pass information back to the application.  This is a one argument function, for instance:

        ```javascript
        function callbackRecorder(x) {
          if (typeof(x.volume) !== 'undefined')
              ...
        }
        ```

        The possible content of the argument are:
	
        * `volume`: Current audio recording volume, between 0 and 100, can be used to show a volume bar in the UI.
        * `volumeMax`: At the end of a recording, it is set to the highest volume encountered during that recording. It is here mainly because Chrome used to have a bug where on several platforms, the recorded audio is silent. So if  `volumeMax` is 0, users should be warned that their setting has the issue.
        * `audio`: A blob that contains the recorded audio, as a `.wav` file. It can be used for example to upload the recording to a server, for later use, for giving a download link to the user, or setting it in the `src` field of an `audio` tag and have users being able to replay their last recording.
* Public methods: The audio recorder just has 2 useful public methods, to start and stop recording:
    * `start(sentence)`: starts recording on the provided sentence.
    * `stop()`: stops recording.

        None of these two methods returns anything, result and errors are given in the previously documented callbacks. For instance, between calls to start and stop, audio recorder callback will be called with the `volume` information, in real time, and after call to stop, audio recorder callback will give the `audio` information with the audio file blob and the WebSocket will receive a message with either the result or an error code.

# 4. Limitations and known issues

## 4.a WebSocket and AudioRecorder initialization order

An application must initialize both a WebSocket and an AudioRecorder. It is not known in advance which one will be initialized first as a connection might take some amount of time to be established and audio recorder is initialized once the user has allowed his browser to access the device. So we advise developers to:

* Create a `websocket` variable, initialized with null, then re-initialized with the WebSocket constructor.
* Give this `websocket` variable to the AudioRecorder constructor.
* Once the `websocket` connection is established, in `onopen`, set the `websocket` again on the `recorder`: `recorder.websocket = websocket`.

## 4.b Browser support

This SDK is working well on recent versions of Google Chrome and Firefox, on all OSes.

## 4.c Firefox scoping hack

As you might see in `wsispikit.html`, we use a hack to keep the input media stream as a workaround for a firefox bug.

```javascript
// input is the result of createMediaStreamSource
// Firefox hack https://support.mozilla.org/en-US/questions/984179
window.firefox_audio_hack = input;
```

## 4.d Secure HTTP issue

For now, the WebSocket server does not use "wss" protocol, so it can not be accessed from a web paged served by "https".