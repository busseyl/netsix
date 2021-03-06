<template>
  <div>
    <form class="form-inline justify-content-center">
      <div class="input-group">
        <input v-model="signalPayload" class="form-control" type="text" placeholder="Enter a WebRTC signal">
        <span class="input-group-btn">
        <button @click="signal()" class="btn btn-primary" type="button" :disabled="signalPayload === ''">Signal</button>
      </span>
      </div>
    </form>

    <form v-if="signalingAnswer" class="form-inline justify-content-center mt-2">
      <div class="input-group">
        <input v-model="signalingAnswer" class="form-control" type="text" disabled>
        <span class="input-group-btn">
        <button v-if="isElectron" @click="copySignal" class="btn btn-primary" type="button">Copy</button>
      </span>
      </div>
    </form>
  </div>
</template>

<script>
  import { mapState } from 'vuex'
  import SimplePeer from 'simple-peer'
  import { clipboard } from 'electron'
  import { bus } from '../../../../shared/bus'

  export default {
    name: 'peer-connection',
    data () {
      return {
        signalPayload: ''
      }
    },
    computed: mapState({
      signalingOffer: state => state.connection.signalingOffer,
      signalingAnswer: state => state.connection.signalingAnswer,
      status: state => state.connection.status,
      isElectron: state => state.configuration.isElectron,
      localCollections: state => state.collections.localCollections
    }),
    created () {
      this.initializePeers()
    },
    methods: {
      initializePeers (canReset) {
        if ((!window.clientPeer && !window.hostPeer) || canReset) {
          this.$store.commit('UPDATE_IS_CONNECTING', true)
          window.clientPeer = new SimplePeer({initiator: true, trickle: false, objectMode: true})
          window.hostPeer = new SimplePeer({trickle: false, objectMode: true})

          // Client peer listeners
          window.clientPeer.on('signal', data => this.handleSignal(window.clientPeer, data))
          window.clientPeer.on('connect', () => this.handleConnect(window.clientPeer))
          window.clientPeer.on('data', data => this.handleData(window.clientPeer, data))
          window.clientPeer.on('close', () => this.handleClose(window.clientPeer))
          window.clientPeer.on('error', err => this.handleError(window.clientPeer, err))

          // Host peer listeners
          window.hostPeer.on('signal', data => this.handleSignal(window.hostPeer, data))
          window.hostPeer.on('connect', () => this.handleConnect(window.hostPeer))
          window.hostPeer.on('data', data => this.handleData(window.hostPeer, data))
          window.hostPeer.on('close', () => this.handleClose(window.hostPeer))
          window.hostPeer.on('error', err => this.handleError(window.hostPeer, err))
        }
      },
      signal () {
        console.log('signal', this.signalPayload)

        let signalPayload = JSON.parse(this.signalPayload)

        if (signalPayload.type === 'offer') {
          window.hostPeer.signal(signalPayload)
        }

        if (signalPayload.type === 'answer') {
          window.clientPeer.signal(signalPayload)
        }
      },
      copySignal () {
        if (this.isElectron) {
          clipboard.writeText(this.signalingAnswer)

          // eslint-disable-next-line no-new
          new Notification('Netsix', {
            body: 'Signal copied to clipboard!'
          })
        }
      },
      handleSignal (peer, data) {
        console.log('PeerConnection: signal: peer, data', peer, data)

        this.$store.commit('UPDATE_IS_CONNECTING', false)

        if (peer.initiator) {
          // when clientPeer has signaling data, give it to hostPeer somehow
          this.$store.commit('UPDATE_SIGNALING_OFFER', JSON.stringify(data)) // window.hostPeer.signal(data)
        } else {
          // when hostPeer has signaling data, give it to clientPeer somehow
          this.$store.commit('UPDATE_SIGNALING_ANSWER', JSON.stringify(data)) // window.clientPeer.signal(data)
        }
      },
      handleConnect (peer) {
        console.log('PeerConnection: connect: peer.initiator', peer.initiator)

        bus.$emit('connection:connect')
        this.$store.commit('UPDATE_IS_CONNECTING', false)
        this.$store.commit('UPDATE_IS_CONNECTED', true)
        this.$store.commit('PUSH_NOTIFICATION', {type: 'success', message: 'You are connected to a peer!'})

        // wait for 'connect' event before using the data channel
        peer.send(JSON.stringify({
          type: 'COLLECTIONS',
          payload: this.localCollections
        }))
      },
      handleData (peer, data) {
        if (typeof data === 'string') {
          try {
            let message = JSON.parse(data)
            console.log('PeerConnection: data: peer.initiator, message', peer.initiator, message)

            switch (message.type) {
              case 'COLLECTIONS':
                // Handle the received collections
                this.$store.commit('UPDATE_REMOTE_COLLECTIONS', Object.assign({}, message.payload))
                break
              case 'CHAT_MESSAGE':
                // Handle the received collections
                this.$store.commit('PUSH_TO_CHATROOM_MESSAGES', {
                  payload: message.payload,
                  from: 'remote'
                })
                break
              case 'GET_FILE_REQUEST':
                // Handle the received file request
                this.$store.dispatch('handleGetFileRequest', message.payload)
                break
              case 'SEND_FILE_INFORMATION':
                // Handle the received file information
                this.$store.commit('UPDATE_SELECTED_FILE', Object.assign({}, message.payload))
                this.$store.commit('UPDATE_REQUESTED_FILE', Object.assign({}, {}))
                break
              case 'ACK_FILE_INFORMATION':
                // Handle the received file acknowledgment
                this.$store.dispatch('handleAckFileInformation', message.payload)
                break
              case 'TRANSCODING_PROGRESS':
                // Handle the received transcoding progress
                this.$store.commit('UPDATE_TRANSCODING_PROGRESS', message.payload)

                // Emit a notification when the transcoding begins/ends
                switch (message.payload) {
                  case 0:
                    this.$store.commit('PUSH_NOTIFICATION', {type: 'info', message: 'Begin to transcode ' + this.$store.state.video.requestedFile.filename + '. It may take a while.'})
                    break
                  case 100:
                    this.$store.commit('PUSH_NOTIFICATION', {type: 'info', message: 'Done transcoding ' + this.$store.state.video.requestedFile.filename + '.'})
                    break
                }
                break
              case 'CANCEL_TRANSCODING_REQUEST':
                // Handle the received cancel transcoding request
                this.$store.dispatch('handleCancelTranscodingInformation')
                break
              case 'NEED_TRANSCODING':
                // Handle the need for transcoding
                this.$store.commit('PUSH_NOTIFICATION', {
                  type: 'warning',
                  message: `Transcoding needed! It may take a while.<br>Video: ${!message.payload.isVideoTypeSupported}, audio: ${!message.payload.isAudioTypeSupported}.`
                })
                break
              default:
                console.warn('PeerConnection: data: unknown type received', message.type)
            }
          } catch (e) {
            // It's a pure string, not a stringified JSON object
            console.error(e, data)
          }
        } else {
          // It's a video chunk
          bus.$emit('video:chunk', data)
        }
      },
      handleClose (peer) {
        console.log('PeerConnection: close: peer.initiator', peer.initiator)
        bus.$emit('connection:close')
        this.$store.commit('UPDATE_IS_CONNECTING', false)
        this.$store.commit('UPDATE_IS_CONNECTED', false)
        this.$store.commit('PUSH_NOTIFICATION', {type: 'danger', message: 'Connection closed!'})
        this.initializePeers(true)
      },
      handleError (peer, err) {
        console.error('PeerConnection: error: peer.initiator, err', peer.initiator, err)
        bus.$emit('connection:error')
        this.$store.commit('UPDATE_IS_CONNECTING', false)
        this.$store.commit('UPDATE_IS_CONNECTED', false)
        this.$store.commit('PUSH_NOTIFICATION', {type: 'danger', message: err.toString()})
        this.initializePeers(true)
      }
    }
  }
</script>
