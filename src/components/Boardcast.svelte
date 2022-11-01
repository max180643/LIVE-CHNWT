<script lang="ts">
  import { onDestroy } from "svelte";
  import { Peer } from "peerjs";
  import { nanoid } from "nanoid";
  import * as mqtt from "mqtt/dist/mqtt.min";

  let mqttClient;
  let mqttTopic = "L1VE@CHNWT";
  let mqttStatus = "Disconnected";
  let liveStatus = "Wait MQTT Server";
  let liveMode = 1;
  let peerList: string[] = [];
  let channelIdInput;
  let channelId: string;
  let mediaStreamData;
  let videoPreviewRef: HTMLVideoElement;

  const peer = new Peer();

  const connectMqtt = () => {
    // connect to MQTT Broker
    mqttClient = mqtt.connect("ws://broker.mqttdashboard.com:8000/mqtt", {
      clientId: `L1VE@CHNWT_${nanoid(10)}`,
    });

    // send streamKey to MQTT Broker
    mqttClient.on("connect", () => {
      console.log("mqtt connect");
      mqttTopic = `${mqttTopic}-${channelId}`;
      mqttClient.subscribe(mqttTopic, function (err) {
        if (err) {
          console.error(err);
          mqttClient.end();
          mqttStatus = "Error";
        }
        mqttStatus = "Connected";
        liveStatus = "Ready";
        mqttClient.publish(mqttTopic, `stream:live`);
        console.log("mqtt send");
      });
    });

    // get message
    mqttClient.on("message", (topic, message) => {
      if (
        topic === mqttTopic &&
        message.toString().includes("stream:") &&
        !message.toString().includes("stream:live")
      ) {
        console.log("mqtt message");
        const msg: string = message.toString();
        const peerId = msg.replace("stream:", "");
        peerList = [...peerList, peerId];
        live(peerId);
      }
    });

    // error
    mqttClient.on("error", (err) => {
      console.error(err);
      mqttStatus = "Error";
      mqttClient.end();
      liveStatus = "Wait MQTT Server";
    });
  };

  const clearVideoPreview = () => {
    videoPreviewRef.currentTime = 0;
    videoPreviewRef.srcObject = null;
  };

  const disconnectMqtt = () => {
    mqttClient.end();
    mqttStatus = "Disconnected";
    liveStatus = "Wait MQTT Server";
    peerList = [];
  };

  const handleLiveButton = async () => {
    channelId = channelIdInput.value;

    if (liveMode === 1) {
      mediaStreamData = await shareScreen();
    } else if (liveMode === 2) {
      mediaStreamData = await shareWebcam();
    } else if (liveMode === 3) {
      mediaStreamData = await shareWebcamNoMic();
    }

    videoPreviewRef.srcObject = mediaStreamData;
    videoPreviewRef.play();

    connectMqtt();
  };

  const handleStopButton = () => {
    clearVideoPreview();
    channelIdInput.value = "D3FAULT";
    channelId = null;
    peer.destroy();
    disconnectMqtt();
    location.reload();
  };

  // share screen
  const shareScreen = async () => {
    let captureStream = null;
    try {
      captureStream = await navigator.mediaDevices.getDisplayMedia({
        video: {
          aspectRatio: 1.77,
          width: { ideal: 1920, max: 1920 },
          height: { ideal: 1080, max: 1080 },
          frameRate: { ideal: 60, max: 60 },
        },
        audio: true,
      });
    } catch (err) {
      console.error(`Error: ${err}`);
    }
    return captureStream;
  };

  // get webcam
  const shareWebcam = async () => {
    let videoStream = null;
    try {
      videoStream = await await navigator.mediaDevices.getUserMedia({
        audio: true,
        video: { width: 1920, height: 1080 },
      });
    } catch (err) {
      console.error(`Error: ${err}`);
    }
    return videoStream;
  };

  // get webcam (no-mic)
  const shareWebcamNoMic = async () => {
    let videoStream = null;
    try {
      videoStream = await await navigator.mediaDevices.getUserMedia({
        audio: false,
        video: { width: 1920, height: 1080 },
      });
    } catch (err) {
      console.error(`Error: ${err}`);
    }
    return videoStream;
  };

  // stream to peer
  const live = (remotePeerId) => {
    peer.call(remotePeerId, mediaStreamData);
  };

  // component unload
  onDestroy(() => {
    if (peer) peer.destroy();
    if (mqttClient) mqttClient.end();
  });
</script>

<div class="container">
  <h1 class="title">Dashboard</h1>
  <div class="panel">
    <div class="left-panel">
      <h2>Config</h2>
      <div>
        <p>Channel ID: {channelId ?? "Not Set"}</p>
        <input bind:this={channelIdInput} value={"D3FAULT"} />
        <button on:click={() => handleLiveButton()}> Live </button>
        <button on:click={() => handleStopButton()}> Stop </button>
      </div>
      <div>
        <p>Live Mode:</p>

        <label>
          <input type="radio" bind:group={liveMode} name="liveMode" value={1} />
          Screen
        </label>

        <label>
          <input type="radio" bind:group={liveMode} name="liveMode" value={2} />
          Webcam
        </label>

        <label>
          <input type="radio" bind:group={liveMode} name="liveMode" value={3} />
          Webcam (No-Mic)
        </label>
      </div>
    </div>
    <div class="right-panel">
      <h2>Status</h2>
      <p>MQTT Server: {mqttStatus}</p>
      <p>Live: {liveStatus}</p>
    </div>
  </div>
  <div class="panel">
    <div class="left-panel">
      <h2>Preview</h2>
      <video bind:this={videoPreviewRef} width="95%" autoplay controls muted>
        <track kind="captions" />
      </video>
    </div>
    <div class="right-panel">
      <h2>Peer List (Total: {peerList.length})</h2>
      {#if peerList}
        <ul>
          {#each peerList as peer}
            <li>{peer}</li>
          {/each}
        </ul>
      {/if}
    </div>
  </div>
  <br />
</div>

<style>
  .container {
    padding-left: 1.5rem;
    padding-right: 1.5rem;
  }

  .title {
    margin-top: 1.5rem;
    margin-bottom: 0rem;
  }

  .panel {
    display: flex;
  }

  .left-panel {
    flex: 1;
  }

  .right-panel {
    flex: 1;
  }
</style>
