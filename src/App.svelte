<script lang="ts">
  const SIGNALING_URL = "34be-89-114-37-188.ngrok.io";
  let ws: WebSocket;
  let polite = false;
  let isInitiator = false;
  let makingOffer = false;

  // check if desktop browser
  if (typeof screen.orientation == "undefined") {
    polite = true;
  }

  const gotMessageFromServer = async (message) => {
    /*
    console.log("got message from server", message.data);

    var signal = JSON.parse(message.data);

    const offerCollision =
      signal.sdp.type == "offer" &&
      (makingOffer || pc.signalingState != "stable");

    if (signal.sdp) {
      await pc.setRemoteDescription(new RTCSessionDescription(signal.sdp));
      if (signal.sdp.type == "offer") {
        isInitiator = false;
        const answer = await pc.createAnswer();
        gotDescription(answer);
      }
    }

    if (signal.ice) pc.addIceCandidate(new RTCIceCandidate(signal.ice));

    */

    const signal = JSON.parse(message.data);
    try {
      if (signal.sdp) {
        const offerCollision =
          signal.sdp.type == "offer" &&
          (makingOffer || pc.signalingState != "stable");

        const ignoreOffer = !polite && offerCollision;
        if (ignoreOffer) {
          return;
        }

        await pc.setRemoteDescription(signal.sdp);
        if (signal.sdp.type == "offer") {
          await pc.setLocalDescription();
          ws.send(JSON.stringify({ sdp: pc.localDescription }));
        }
      } else if (signal.ice) {
        await pc.addIceCandidate(signal.ice);
      }
    } catch (err) {
      console.error(err);
    }
  };

  const setupWebsocketConnection = () => {
    ws = new WebSocket(`wss://${SIGNALING_URL}`);
    ws.onmessage = gotMessageFromServer;

    ws.onclose = setupWebsocketConnection;
  };

  setupWebsocketConnection();

  /*
  const gotDescription = async (description: RTCSessionDescriptionInit) => {
    await pc.setLocalDescription(description);
    console.log("sending description to signaling server");
    ws.send(JSON.stringify({ sdp: description }));
  };
*/

  let pc: RTCPeerConnection;

  pc.onnegotiationneeded = async () => {
    try {
      makingOffer = true;
      await pc.setLocalDescription();
      ws.send(JSON.stringify({ sdp: pc.localDescription }));
    } catch (err) {
      console.error(err);
    } finally {
      makingOffer = false;
    }
  };

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

  pc = new RTCPeerConnection(peerConnectionConfig);
 
  /*
  const start = async () => {
    isInitiator = true;
    const offer = await pc.createOffer();
    gotDescription(offer);
  };
  */

  const gotIceCandidate = (event) => {
    if (event.candidate != null) {
      ws.send(JSON.stringify({ ice: event.candidate }));
    }
  };

  pc.onicecandidate = gotIceCandidate;

  const dataChannel = pc.createDataChannel("data", {
    negotiated: true,
    id: 0,
    ordered: true,
  });

  let receivedMessage = "";

  dataChannel.onmessage = (ev) => {
    console.log("got message: ", ev.data);
    receivedMessage = ev.data;
  };

  let sendBtnActivated = false;

  dataChannel.onopen = () => {
    console.log("dataChannel open");
    sendBtnActivated = true;
  };

  dataChannel.onclose = () => {
    console.log("dataChannel closed");
    sendBtnActivated = false;
  };

  pc.addEventListener("iceconnectionstatechange", (event) => {
    console.log("iceConnectionState", pc.iceConnectionState);
    if (pc.iceConnectionState === "failed") {
      /* possibly reconfigure the connection in some way here */
      /* then request ICE restart */
      console.log("ice connection state: FAILED");
      pc.restartIce();
    }
  });

  let message;

  const sendMessage = () => dataChannel.send(message);
</script>

<input type="text" bind:value={message} />
<button on:click={sendMessage} disabled={sendBtnActivated}>Send</button>
Got message: {receivedMessage}
