<template>
  <div class="background">
    <div class="text-center text-white" style="padding-top: 50px; width: 100%;">
      <p class="display-1 fw-bold">Test WebRTC</p>
      <p class="fs-1 fw-bold">for werewolf web game project</p>
    </div>
    <div id="chat-container" class="chat-container">
        <div id="left" class="split">
            <video autoplay id="user-video" class="user-video"></video>
            <div class="controls">
              <button class="button" style="background-color: transparent;" id="toggle-mic" v-on:click="toggleMic">
                <img v-if="state" src="https://www.markhillard.com/sandbox/micon.png" alt="microphone is 'on'">
                <img v-else style="color: #FF0000;" src="https://www.markhillard.com/sandbox/micoff.png" alt="microphone is 'off'">
              </button>
            </div>
        </div>
        <div id="right" class="split"></div>
    </div>
  </div>
</template>

<script>
import io from "socket.io-client";
export default {
  data() {
    return {
      socket: {},

      peers: {},
      userStream: null,
      state: false,
      currentUserContainer: null,
      currentUserVideo: null,
      otherUserVideo: null,
    }
  },
  async created() {
    this.socket = io('http://localhost:1337');
    console.log(this.socket)

    this.init();
  },
  mounted() {

  },
  methods: {
    async init() {
      this.socket.on('connect', async () => {
        let stream = await navigator.mediaDevices.getUserMedia({ audio: true, video: true });
        this.userStream = stream;
        this.currentUserVideo = document.getElementById('user-video');
        this.currentUserVideo.srcObject = stream;

        this.currentUserContainer = document.getElementById('left');
        this.otherUserVideo = document.getElementById('right');

        let vid = this.userStream.getTracks().find(track => track.kind === 'video');
        let mic = this.userStream.getTracks().find(track => track.kind === 'audio');
        vid.enabled = false;
        mic.enabled = false;

        let roomId = window.location.pathname.split('/')[2]

        this.socket.emit('user joined room', roomId);

        this.socket.on('all other users', (otherUsers) => this.callOtherUsers(otherUsers, this.userStream));

        this.socket.on("connection offer", (payload) => this.handleReceiveOffer(payload, this.userStream));

        this.socket.on('connection answer', this.handleAnswer);

        this.socket.on('ice-candidate', this.handleReceiveIce);
      });
    },
    callOtherUsers(otherUsers, stream) {
      otherUsers.forEach(userIdToCall => {
        const peer = this.createPeer(userIdToCall);
        this.peers[userIdToCall] = peer;
        stream.getTracks().forEach(track => {
            peer.addTrack(track, stream);
        });
      });
    },
    createPeer(userIdToCall) {
      const peer = new RTCPeerConnection({
          iceServers: [
              {
                  urls: "stun:stun.stunprotocol.org"
              }
          ]
      });
      peer.onnegotiationneeded = () => userIdToCall ? this.handleNegotiationNeededEvent(peer, userIdToCall) : null;
      peer.onicecandidate = this.handleICECandidateEvent;
      peer.ontrack = (e) => {
          const container = document.createElement('div');
          container.classList.add('remote-video-container');
          const video = document.createElement('video');
          video.srcObject = e.streams[0];
          video.autoplay = true;
          video.playsInline = true;
          video.classList.add("remote-video");
          container.appendChild(video);
          container.id = userIdToCall;
          this.otherUserVideo.appendChild(container);
      }
      return peer;
    },
    async handleNegotiationNeededEvent(peer, userIdToCall) {
      const offer = await peer.createOffer();
      await peer.setLocalDescription(offer);
      const payload = {
          sdp: peer.localDescription,
          userIdToCall,
      };

      this.socket.emit('peer connection request', payload);
    },
    async handleReceiveOffer({ sdp, callerId }, stream) {
      const peer = this.createPeer(callerId);
      this.peers[callerId] = peer;
      const desc = new RTCSessionDescription(sdp);
      await peer.setRemoteDescription(desc);

      stream.getTracks().forEach(track => {
        peer.addTrack(track, stream);
      });

      const answer = await peer.createAnswer();
      await peer.setLocalDescription(answer);

      const payload = {
        userToAnswerTo: callerId,
        sdp: peer.localDescription,
      };

      this.socket.emit('connection answer', payload);
    },
    handleAnswer({ sdp, answererId }) {
      const desc = new RTCSessionDescription(sdp);
      this.peers[answererId].setRemoteDescription(desc).catch(e => console.log(e));
    },
    handleICECandidateEvent(e) {
      if (e.candidate) {
        Object.keys(this.peers).forEach(id => {
          const payload = {
            target: id,
            candidate: e.candidate,
          }
          this.socket.emit("ice-candidate", payload);
        });
      }
    },
    handleReceiveIce({ candidate, from }) {
      const inComingCandidate = new RTCIceCandidate(candidate);
      this.peers[from].addIceCandidate(inComingCandidate);
    },
    async toggleMic() {
        let videoTrack = this.userStream.getTracks().find(track => track.kind === 'video');
        let audioTrack = this.userStream.getTracks().find(track => track.kind === 'audio');
        if (videoTrack.enabled && audioTrack.enabled) {
        // if (audioTrack.enabled) {
          videoTrack.enabled = false;
          audioTrack.enabled = false;
          this.state = false;
        } else {
          videoTrack.enabled = true;
          audioTrack.enabled = true;
          this.state = true;
        }
    },
  }
}
</script>

<style scoped>
.background{
  background-color: #333333;
  height: 100%;
  text-align: right;
}
input::-webkit-input-placeholder{
    color:rgb(172, 172, 172);
}
.m-title-wrapper{
  border-bottom: 2px solid white;
}

.chat-container {
    height: 100%;
    width: 100%;
    display: flex;
}

/* .split {
    width: 400px;
    height: 300px;
    display: flex;
    flex-direction: column;
    justify-content: center;
} */

.user-video {
    height: auto;
    width: 100%;
}

.controls {
    width: 90%;
    padding: 12px;
    display: flex;
    box-sizing: border-box;
    display: flex;
    justify-content: center;
}

.remote-video {
    width: 100%;
    height: auto;
}

.remote-video-container {
    width: 50%;
    display: flex;
    flex-direction: column;
}
</style>