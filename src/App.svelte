<script lang="ts">
  const SIGNALING_URL = "34be-89-114-37-188.ngrok.io";
  let serverConnection: WebSocket;

  let isInitiator = false;

  const gotMessageFromServer = async (message) => {
    console.log("got message from server", message.data);

    var signal = JSON.parse(message.data);

    if (signal.sdp) {
      await peerConnection.setRemoteDescription(
        new RTCSessionDescription(signal.sdp)
      );
      if (signal.sdp.type == "offer") {
        isInitiator = false;
        const answer = await peerConnection.createAnswer();
        gotDescription(answer);
      }
    }

    if (signal.ice)
      peerConnection.addIceCandidate(new RTCIceCandidate(signal.ice));
  };

  const setupWebsocketConnection = () => {
    serverConnection = new WebSocket(`wss://${SIGNALING_URL}`);
    serverConnection.onmessage = gotMessageFromServer;

    serverConnection.onclose = () => {
      serverConnection = new WebSocket(`wss://${SIGNALING_URL}`);
      serverConnection.onmessage = gotMessageFromServer;
    };
  };

  setupWebsocketConnection();

  const gotDescription = async (description: RTCSessionDescriptionInit) => {
    await peerConnection.setLocalDescription(description);
    console.log("sending description to signaling server");
    serverConnection.send(JSON.stringify({ sdp: description }));
  };

  let peerConnection: RTCPeerConnection;

  const peerConnectionConfig = {
    iceServers: [
      {
        urls: "stun:stun.stunprotocol.org",
      },
      {
        urls: "turn:k8s-signalin-turnserv-46e371bfb0-ed89a06c3bcabcd8.elb.eu-west-2.amazonaws.com:80",
        username: "username",
        credential: "password",
      },
      // {
      //   urls: "stun:stun.stunprotocol.org",
      // },
      // {
      //   urls: "turn:openrelay.metered.ca:80",
      //   username: "openrelayproject",
      //   credential: "openrelayproject",
      // },
      // {
      //   urls: "turn:signaling-server-pr-12.rdx-works-main.extratools.works:80",
      //   username: "username",
      //   credential: "password",
      // },
    ],
  };

  peerConnection = new RTCPeerConnection(peerConnectionConfig);

  const start = async () => {
    isInitiator = true;
    const offer = await peerConnection.createOffer();
    gotDescription(offer);
  };

  const gotIceCandidate = (event) => {
    if (event.candidate != null) {
      serverConnection.send(JSON.stringify({ ice: event.candidate }));
    }
  };

  peerConnection.onicecandidate = gotIceCandidate;

  const dataChannel = peerConnection.createDataChannel("data", {
    negotiated: true,
    id: 0,
    ordered: true,
  });

  let receivedMessage = "";

  dataChannel.onmessage = (ev) => {
    console.log("got message: ", ev.data);
    receivedMessage = ev.data;
  };

  dataChannel.onopen = () => {
    console.log("dataChannel open");
  };

  dataChannel.onclose = () => {
    console.log("dataChannel closed");
  };

  peerConnection.addEventListener("iceconnectionstatechange", (event) => {
    console.log("iceConnectionState", peerConnection.iceConnectionState);
    if (peerConnection.iceConnectionState === "failed") {
      /* possibly reconfigure the connection in some way here */
      /* then request ICE restart */
      console.log("ice connection state: FAILED");
      peerConnection.restartIce();
      if (isInitiator) start();
    }
  });

  let message;

  const sendMessage = () => dataChannel.send(message);
</script>

<button on:click={start}>Start</button>
<input type="text" bind:value={message} />
<button on:click={sendMessage}>Send</button>
Got message: {receivedMessage}
