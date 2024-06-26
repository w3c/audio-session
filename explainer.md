# Audio Session API Explainer

## Objectives
People consume a lot of media (audio/video) and the Web is one of the primary means of consuming this type of content.
However, media on the web does not integrate well with the platform.
The Audio Session API helps to close the gap with platforms that have audio session/audio focus such as Android and iOS.
This API will help by improving the audio-mixing of websites with native apps, so they can play on top of each other, or play exclusively.

Additionally, on some platforms the user agent will automatically manage the audio session for the site based on whether media elements are playing or not and which APIs are used for playing audio.
In some cases this may not match user expectations so this API will provide overrides for the authors.

### Goals

 * **A site should be able to define how audio streams will interact with the platform.**
   This is where it can be annoying where two tabs play audio at the same time.
   However, in some cases it may be appropriate to play the two audio streams on top of each other (e.g. a transient ping).
 * **A site should be able to manage its own audio session and focus.**
   If a site wishes to manage its own audio focus (when to restart playing once uninterrupted for instance) then the user agent should not automatically manage it.
   This would be used on a site where the default user agent audio focus logic is not appropriate (a media site where we switch tracks) or supported (e.g. WebAudio, WebRTC sites).
 * **A site should be able to determine its own audio session state.**
   A site should be notified if its audio session state changes.
   This is so sites that are manually managing focus can be aware of their current state.
 * **To provide an experience on par with native apps.**
   Native apps have audio session APIs on some platforms so we should provide a similar level of experience for websites.

### Non-goals

* We should aim to improve the audio interaction between the site and the platform. Audio interaction within the site should be up to the site in the first version of this document.  For instance, a web site cannot manipulate more than one audio session and cannot group audio producers in different audio sesssions.

## API Design

By convention, there are several `audio session types` for different purposes:

 * Playback (`playback`) audio, which is used for video or music playback, podcasts, etc. They should not mix with other playback audio. (Maybe) they should pause all other audio indefinitely.
 * Transient (`transient`) audio, such as a notification ping. They usually should play on top of playback audio (and maybe also "duck" persistent audio).
 * Transient solo (`transient-solo`) audio, such as driving directions. They should pause/mute all other audio and play exclusively. When a transient-solo audio ended, it should resume the paused/muted audio.
 * Ambient (`ambient`) audio, which is mixable with other types of audio. This is useful in some special cases such as when the user wants to mix audios from multiple pages.
 * Play and record (`play-and-record`) audio, which is used for recording audio. This is useful in cases microphone is being used or in video conferencing applications.
 * Auto (`auto`) lets the User Agent choose the best audio session type according the use of audio by the web page. This is the type of the default AudioSession.

The AudioSession is the main interface for this API. It can have the following states:

 * active: the AudioSession is playing sound.
 * interrupted: the AudioSession is not playing sound, but can resume when it will get uninterrupted.
 * inactive: the AudioSession is not playing sound.

The page has a default audio session which is used by the user agent to automatically set up the audio session parameters.
The UA will request and abandon audio focus when media elements start/finish playing on the page.
This default audio session is represented as an `AudioSession` object that is exposed as `navigator.audioSession`.

```javascript
enum AudioSessionState {
  "inactive",
  "active",
  "interrupted"
};

enum AudioSessionType {
  "auto",
  "playback",
  "transient",
  "transient-solo",
  "ambient",
  "play-and-record"
};

[Exposed=Window]
partial interface Navigator {
  // The default audio session that the user agent will use when media elements start/stop playing.
  readonly attribute AudioSession audioSession;
};

[Exposed=Window]
interface AudioSession : EventTarget {
  attribute AudioSessionType type;

  readonly attribute AudioSessionState state;
  attribute EventHandler onstatechange;
};
```

## Sample Code

#### A site sets its audio session type proactively to "play-and-record"

```javascript
navigator.audioSession.type = 'play-and-record';
// From now on, volume might be set based on 'play-and-record'.
...
// Start playing remote media
remoteVideo.srcObject = remoteMediaStream;
remoteVideo.play();
// Start capturing
navigator.mediaDevices.getUserMedia({ audio:true, video:true }).then(stream => {
    localVideo.srcObject = stream;
});
```

#### A site reacts upon interruption

```javascript
navigator.audioSession.type = 'play-and-record';
// From now on, volume might be set based on 'play-and-record'.
...
// Start playing remote media
remoteVideo.srcObject = remoteMediaStream;
remoteVideo.play();
// Start capturing
navigator.mediaDevices.getUserMedia({ audio:true, video:true }).then(stream => {
    localVideo.srcObject = stream;
});

let isInterrupted = false;
navigator.audioSession.onstatechange = () => {
    if (navigator.audioSession.state === 'interrupted') {
        isInterrupted = true;
        localVideo.pause();
        remoteVideo.pause();
        // Make it clear to the user that the call is interrupted.
        showInterruptedBanner();
        localVideo.srcObject.getTracks().forEach(track => track.enabled = false);
        return;
    }
    if (isInterrupted) {
        isInterrupted = false;
        // Let user decide when to restart the call.
        showOptionalRestartBanner().then((result) => {
            if (!result)
                return;
            localVideo.srcObject.getTracks().forEach(track => track.enabled = true);
            localVideo.play();
            remoteVideo.play();
        });
    }
}
```
