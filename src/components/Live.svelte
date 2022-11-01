<script lang="ts">
  import { onMount, onDestroy } from "svelte";
  import { Peer } from "peerjs";
  import { nanoid } from "nanoid";
  import * as mqtt from "mqtt/dist/mqtt.min";

  let peer: Peer;
  let mqttClient;
  let videoStreamRef;
  let mqttTopic = "L1VE@CHNWT";
  let streamKey = "";
  let channelId: string;
  let state = "OFFLINE";

  onMount(async () => {
    const path = window.location.pathname;
    const page = path.substring(1).toLowerCase();

    if (page === "") {
      // get channel id from param
      let url = new URL(window.location.href);
      channelId = url.searchParams.get("c");

      // set mqtt topic
      if (!channelId) {
        mqttTopic = `${mqttTopic}-D3FAULT`;
      } else {
        mqttTopic = `${mqttTopic}-${channelId}`;
      }

      // get peer id
      peer = new Peer();

      peer.on("open", async (peerId) => {
        console.log("peer open");
        streamKey = peerId;

        // connect to MQTT Broker
        mqttClient = mqtt.connect("wss://broker.emqx.io:8084/mqtt", {
          clientId: `L1VE@CHNWT_${nanoid(10)}`,
        });

        // send streamKey to MQTT Broker
        mqttClient.on("connect", () => {
          console.log("mqtt connect");

          const msg = `stream:${streamKey}`;
          mqttClient.publish(mqttTopic, msg);
          // subscribe topic
          mqttClient.subscribe(mqttTopic, function (err) {
            if (err) {
              console.error(err);
              mqttClient.end();
            } else {
              // get message
              mqttClient.on("message", (topic, message) => {
                if (
                  topic === mqttTopic &&
                  message.toString().includes("stream:live")
                ) {
                  console.log("mqtt message");
                  mqttClient.publish(mqttTopic, msg);
                  console.log("mqtt send");
                }
              });
            }
          });
        });
      });

      // handle connection
      peer.on("call", async (call) => {
        // accept connection
        call.answer();
        // render stream
        state = "LIVE";
        call.on("stream", renderStream);
        console.log("live");
      });

      // handle close
      peer.on("close", async () => {
        state = "OFFLINE";
        peer.destroy();
        console.log("peer close");
      });

      // handle disconnected
      peer.on("disconnected", async () => {
        state = "OFFLINE";
        peer.reconnect();
        console.log("peer disconnected");
      });

      // handle error
      peer.on("error", async (err) => {
        state = "OFFLINE";
        console.log(err);
      });
    }
  });

  // render stream
  const renderStream = (mediaStream) => {
    videoStreamRef.srcObject = mediaStream;
    videoStreamRef.play();
  };

  // component unload
  onDestroy(() => {
    if (peer) peer.destroy();
    if (mqttClient) mqttClient.end();
  });
</script>

<div>
  {#if state === "OFFLINE"}
    <div class="container-center">
      <h3>THE STREAM IS CURRENTLY</h3>
      <h1>OFFLINE</h1>
      <h3>LIVE @ CHNWT</h3>
    </div>
  {:else if state === "LIVE"}
    <video
      bind:this={videoStreamRef}
      poster="cover.png"
      controls
      autoplay
      muted
    >
      <track kind="captions" />
    </video>
  {/if}
</div>

<style>
  .container-center {
    height: 100vh;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
  }

  video {
    height: 100vh;
    width: 100%;
    object-fit: cover;
    position: absolute;
  }
</style>
