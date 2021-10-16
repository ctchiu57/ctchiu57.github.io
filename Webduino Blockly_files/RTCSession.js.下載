+(function (global) {

  'use strict';

  var RTCMultiConnection = global.RTCMultiConnection,
    RTCPeerConnection = global.RTCPeerConnection || global.mozRTCPeerConnection || global.webkitRTCPeerConnection,
    RTCSessionDescription = global.RTCSessionDescription || global.mozRTCSessionDescription,
    RTCIceCandidate = global.RTCIceCandidate || global.mozRTCIceCandidate,
    getUserMedia = global.navigator.getUserMedia || global.navigator.mozGetUserMedia || global.navigator.webkitGetUserMedia,
    URL = global.URL;

  function RTCSession(roomId, password, captureUserMedia) {
    var connection = this.connection = new RTCMultiConnection(roomId);
    var self = this;

    connection.session = {
      audio: true,
      video: true
    };

    connection.extra = {
      password: password || ''
    };

    connection.dontCaptureUserMedia = !(captureUserMedia || false);

    connection.onNewSession = function (session) {
      // console.log('new session: ' + session.sessionid);
      session.join();
    };

    connection.onRequest = function (e) {
      if (e.extra.password !== password) {
        connection.reject(e);
        return;
      }
      connection.accept(e);
    };

    connection.onstream = function (event) {
      if (typeof self.onstream === 'function') {
        self.onstream(event);
      }
    };

    connection.onstreamended = function (event) {
      if (typeof self.onstreamended === 'function') {
        self.onstreamended(event);
      }
    };

    connection.onstatechange = function (event) {
      if (typeof self.onstatechange === 'function') {
        self.onstatechange(event);
      }
    };

    connection.connect();
  }

  RTCSession.prototype.open = function () {
    this.connection.open();
  };

  RTCSession.prototype.attachRemote = function (address) {
    this.remote = new WebSocketSignalling(address, this.connection);
  };

  RTCSession.prototype.close = function () {
    if (this.remote) {
      this.remote.stop();
    }
    this.connection.leave();
    this.connection.disconnect();
  };

  var pcConfig = {
      "iceServers": [{
        url: "stun:stun.l.google.com:19302"
      }]
    },
    pcOptions = {
      optional: [{
        DtlsSrtpKeyAgreement: true
      }]
    },
    mediaConstraints = {
      optional: [],
      mandatory: {
        OfferToReceiveAudio: true,
        OfferToReceiveVideo: true
      }
    };

  function WebSocketSignalling(address, connection) {
    var ws = this.ws = new WebSocket(address);
    var self = this;

    self.connection = connection;

    ws.onopen = function () {
      // console.log("onopen()");
      createPeerConnection(self);
      var command = {
        command_id: "offer"
      };
      ws.send(JSON.stringify(command));
      // console.log("onopen(), command=" + JSON.stringify(command));
    };

    ws.onmessage = function (evt) {
      var pc = self.pc;
      var msg = JSON.parse(JSON.parse(evt.data).data);
      //console.log("message=" + msg);
      // console.log("type=" + msg.type);

      switch (msg.type) {
      case "offer":
        pc.setRemoteDescription(new RTCSessionDescription(msg),
          function onRemoteSdpSuccess() {
            // console.log('onRemoteSdpSucces()');
            pc.createAnswer(function (sessionDescription) {
              pc.setLocalDescription(sessionDescription);
              var command = {
                command_id: "answer",
                data: JSON.stringify(sessionDescription)
              };
              ws.send(JSON.stringify(command));
              // console.log(command);

            }, function (error) {
              alert("Failed to createAnswer: " + error);

            }, mediaConstraints);
          },
          function onRemoteSdpError(event) {
            alert('Failed to setRemoteDescription: ' + event);
          }
        );

        var command = {
          command_id: "geticecandidate"
        };
        // console.log(command);
        ws.send(JSON.stringify(command));
        break;

      case "answer":
        break;

      case "message":
        alert(msg.data);
        break;

      case "geticecandidate":
        var candidates = JSON.parse(msg.data);
        for (var i = 0; i < candidates.length; i++) {
          var elt = candidates[i];
          var candidate = new RTCIceCandidate({
            sdpMLineIndex: elt.sdpMLineIndex,
            candidate: elt.candidate
          });
          pc.addIceCandidate(candidate,
            function () {
              // console.log("IceCandidate added: " + JSON.stringify(candidate));
            },
            function (error) {
              // console.log("addIceCandidate error: " + error);
            }
          );
        }
        break;
      }
    };

    ws.onclose = function (evt) {
      var pc = self.pc;

      if (pc) {
        pc.close();
        self.pc = pc = null;
      }
    };

    ws.onerror = function (evt) {
      alert("An error has occurred!");
      ws.close();
    };
  }

  function createPeerConnection(self) {
    try {
      var pc = self.pc = new RTCPeerConnection(pcConfig, pcOptions);
      pc.onicecandidate = onIceCandidate.bind(self);
      pc.onaddstream = onRemoteStreamAdded.bind(self);
      pc.onremovestream = onRemoteStreamRemoved.bind(self);
      // console.log("peer connection successfully created!");
    } catch (e) {
      // console.log("createPeerConnection() failed");
    }
  }

  function onIceCandidate(event) {
    if (event.candidate) {
      var candidate = {
        sdpMLineIndex: event.candidate.sdpMLineIndex,
        sdpMid: event.candidate.sdpMid,
        candidate: event.candidate.candidate
      };
      var command = {
        command_id: "addicecandidate",
        data: JSON.stringify(candidate)
      };
      this.ws.send(JSON.stringify(command));
    } else {
      // console.log("End of candidates.");
    }
  }

  function onRemoteStreamAdded(event) {
    var connection = this.connection;

    connection.attachExternalStream(event.stream, false);
    if (!connection.isInitiator)
      connection.renegotiate();
  }

  function onRemoteStreamRemoved(event) {
    this.connection.removeStream(event.stream);
  }

  WebSocketSignalling.prototype.stop = function () {
    var ws = this.ws,
      pc = this.pc;

    if (pc) {
      pc.close();
      this.pc = pc = null;
    }
    if (ws) {
      ws.close();
      this.ws = ws = null;
    }
  };

  global.RTCSession = RTCSession;

}(window));
