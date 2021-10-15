<template>
  <main>
    <div class="container" style="text-align:center;">
      <div class="row">
        <div class="col">
          <div class="btn-group" role="group">
            <button
              type="button"
              class="btn mr-2 btn-mn"
              v-for="user in allusers"
              :key="user.id"
              @click="placeCall(user.id, user.name)"
            >
              Call {{ user.name }}
              <span v-if="getUserOnlineStatus(user.id) === 'Online'" class="online"></span>
              <span v-if="getUserOnlineStatus(user.id) === 'Offline'" class="offline"></span>
            </button>
          </div>
        </div>
      </div>

      <!-- Incoming Call  -->
      <div v-if="incomingCall" style="
          position: absolute;
          bottom: 0px;
          left: 0px;
          width: calc(100% - 30px);
          text-align: center;
          background: #fff;
          border: 1px solid #eee;
          margin: 15px 15px 60px 15px;
          padding: 20px 0 25px 0;">
        <div class="row">
          <div class="col-12">
            <p>
              Incoming Call From <strong>{{ incomingCaller }}</strong>
            </p>
            <div class="btn-group" role="group">
              <button
                type="button"
                class="btn btn-danger"
                data-dismiss="modal"
                @click="declineCall"
              >
                Decline
              </button>
              <button
                type="button"
                class="btn btn-success"
                @click="acceptCall"
              >
                Accept
              </button>
            </div>
          </div>
        </div>
      </div>
      <!-- End of Incoming Call  -->
    </div>

    <section id="video-container" v-if="callPlaced">
      <div id="local-video"></div>
      <div id="remote-video"></div>

      <div class="action-btns">
        <button type="button" class="btn btn-default" @click="handleAudioToggle">
          <i class="fa fa-microphone-slash" v-if="mutedAudio"></i>
          <i class="fa fa-microphone" v-if="!mutedAudio"></i>
        </button>
        <button type="button" class="btn btn-danger" @click="endCall">
          <i class="fa fa-phone"></i>
        </button>
        <button
          type="button"
          class="btn btn-default"
          @click="handleVideoToggle"
        >
          <i class="fa fa-video-camera" v-if="mutedVideo"></i>
          <i class="fa fa-video-camera" v-if="!mutedVideo"></i>
        </button>
      </div>
    </section>
  </main>
</template>

<script>
export default {
  name: "AgoraChat",
  props: ["authuser", "authuserid", "allusers", "agora_id"],
  data() {
    return {
      callPlaced: false,
      client: null,
      localStream: null,
      mutedAudio: false,
      mutedVideo: false,
      userOnlineChannel: null,
      onlineUsers: [],
      incomingCall: false,
      incomingCaller: "",
      agoraChannel: null,
    };
  },

  mounted() {
    this.initUserOnlineChannel();
    this.initUserOnlineListeners();
  },

  methods: {
    /**
     * Presence Broadcast Channel Listeners and Methods
     * Provided by Laravel.
     * Websockets with Pusher
     */
    initUserOnlineChannel() {
      this.userOnlineChannel = window.Echo.join("agora-online-channel");
    },

    initUserOnlineListeners() {
      this.userOnlineChannel.here((users) => {
        this.onlineUsers = users;
      });

      this.userOnlineChannel.joining((user) => {
        // check user availability
        const joiningUserIndex = this.onlineUsers.findIndex(
          (data) => data.id === user.id
        );
        if (joiningUserIndex < 0) {
          this.onlineUsers.push(user);
        }
      });

      this.userOnlineChannel.leaving((user) => {
        const leavingUserIndex = this.onlineUsers.findIndex(
          (data) => data.id === user.id
        );
        this.onlineUsers.splice(leavingUserIndex, 1);
      });

      // listen to incomming call
      this.userOnlineChannel.listen("MakeAgoraCall", ({ data }) => {
        if (parseInt(data.userToCall) === parseInt(this.authuserid)) {
          const callerIndex = this.onlineUsers.findIndex(
            (user) => user.id === data.from
          );
          this.incomingCaller = this.onlineUsers[callerIndex]["name"];
          this.incomingCall = true;

          // the channel that was sent over to the user being called is what
          // the receiver will use to join the call when accepting the call.
          this.agoraChannel = data.channelName;
        }
      });
    },

    getUserOnlineStatus(id) {
      const onlineUserIndex = this.onlineUsers.findIndex(
        (data) => data.id === id
      );
      if (onlineUserIndex < 0) {
        return "Offline";
      }
      return "Online";
    },

    async placeCall(id, calleeName) {
      try {
        // channelName = the caller's and the callee's id. you can use anything. tho.
        const channelName = `${this.authuser}_${calleeName}`;
        const tokenRes = await this.generateToken(channelName);

        // Broadcasts a call event to the callee and also gets back the token
        await axios.post("/agora/call-user", {
          user_to_call: id,
          username: this.authuser,
          channel_name: channelName,
        });

        this.initializeAgora();
        this.joinRoom(tokenRes.data, channelName);
      } catch (error) {
        console.log(error);
      }
    },

    async acceptCall() {
      this.initializeAgora();
      const tokenRes = await this.generateToken(this.agoraChannel);
      this.joinRoom(tokenRes.data, this.agoraChannel);
      this.incomingCall = false;
      this.callPlaced = true;
    },

    declineCall() {
      // You can send a request to the caller to
      // alert them of rejected call
      this.incomingCall = false;
    },

    generateToken(channelName) {
      return axios.post("/agora/token", {
        channelName,
      });
    },

    /**
     * Agora Events and Listeners
     */
    initializeAgora() {
      this.client = AgoraRTC.createClient({ mode: "rtc", codec: "h264" });
      this.client.init(
        this.agora_id,
        () => {
          console.log("AgoraRTC client initialized");
        },
        (err) => {
          console.log("AgoraRTC client init failed", err);
        }
      );
    },

    async joinRoom(token, channel) {
      this.client.join(
        token,
        channel,
        this.authuser,
        (uid) => {
          console.log("User " + uid + " join channel successfully");
          this.callPlaced = true;
          this.createLocalStream();
          this.initializedAgoraListeners();
        },
        (err) => {
          console.log("Join channel failed", err);
        }
      );
    },

    initializedAgoraListeners() {
      //   Register event listeners
      this.client.on("stream-published", function (evt) {
        console.log("Publish local stream successfully");
        console.log(evt);
      });

      //subscribe remote stream
      this.client.on("stream-added", ({ stream }) => {
        console.log("New stream added: " + stream.getId());
        this.client.subscribe(stream, function (err) {
          console.log("Subscribe stream failed", err);
        });
      });

      this.client.on("stream-subscribed", (evt) => {
        // Attach remote stream to the remote-video div
        evt.stream.play("remote-video");
        this.client.publish(evt.stream);
      });

      this.client.on("stream-removed", ({ stream }) => {
        console.log(String(stream.getId()));
        stream.close();
      });

      this.client.on("peer-online", (evt) => {
        console.log("peer-online", evt.uid);
      });

      this.client.on("peer-leave", (evt) => {
        var uid = evt.uid;
        var reason = evt.reason;
        console.log("remote user left ", uid, "reason: ", reason);
      });

      this.client.on("stream-unpublished", (evt) => {
        console.log(evt);
      });
    },

    createLocalStream() {
      this.localStream = AgoraRTC.createStream({
        audio: true,
        video: true,
      });

      // Initialize the local stream
      this.localStream.init(
        () => {
          // Play the local stream
          this.localStream.play("local-video");
          // Publish the local stream
          this.client.publish(this.localStream, (err) => {
            console.log("publish local stream", err);
          });
        },
        (err) => {
          console.log(err);
        }
      );
    },

    endCall() {
      this.localStream.close();
      this.client.leave(
        () => {
          console.log("Leave channel successfully");
          this.callPlaced = false;
        },
        (err) => {
          console.log("Leave channel failed");
        }
      );
    },

    handleAudioToggle() {
      if (this.mutedAudio) {
        this.localStream.unmuteAudio();
        this.mutedAudio = false;
      } else {
        this.localStream.muteAudio();
        this.mutedAudio = true;
      }
    },

    handleVideoToggle() {
      if (this.mutedVideo) {
        this.localStream.unmuteVideo();
        this.mutedVideo = false;
      } else {
        this.localStream.muteVideo();
        this.mutedVideo = true;
      }
    },
  },
};
</script>

<style scoped>
@import "https://stackpath.bootstrapcdn.com/bootstrap/4.1.2/css/bootstrap.min.css";
main {
  margin-top: 50px;
}

#video-container {
  max-width: 100%;
  max-height: 100%;
  width: 100%;
  height: 100%;
  margin: 0 auto;
  position: absolute;
  background-color: #fff;
  top: 0;
  left: 0;
  z-index: 1;
}

#local-video {
  width: 30%;
  height: 30%;
  position: absolute;
  right: 0;
  top: 0;
  border: 1px solid #fff;
  z-index: 2;
  cursor: pointer;
  margin: 15px 15px 0 0;
}

#remote-video {
  width: 100%;
  height: 100%;
  position: absolute;
  left: 0;
  right: 0;
  bottom: 0;
  top: 0;
  z-index: 1;
  margin: 0;
  padding: 0;
  cursor: pointer;
}

.action-btns {
  position: absolute;
  bottom: 20px;
  z-index: 3;
  text-align: center;
  width: 100%;
}

.action-btns button {
  padding: 20px 25px;
  border-radius: 100%;
}

.action-btns button.btn-default {
  background: #aaa;
}

#login-form {
  margin-top: 100px;
}

.btn-mn {
	background-color: #774e9d;
  color: #fff;
  display: block;
  padding: 10px 25px;
}

.btn-mn span {
  display: inline-block;
  width: 10px;
  height: 10px;
  border-radius: 100%;
  margin-bottom: -1px;
  margin-left: 5px;
}
.btn-mn span.offline {
  color: #ddd;
  background-color: #ddd;
}
.btn-mn span.online {
  color: #ddd;
  background-color: green;
}
</style>