<template>
  <div>
    <h1>Simple Peer - Audio Call</h1>
    <p>Your ID: {{ myId }}</p>
    <input v-model="targetId" placeholder="Enter Target ID" />
    <button @click="startCall">Start Call</button>
    <button @click="endCall">End Call</button>

    <!-- Mostrar el audio local -->
    <audio ref="localAudio" autoplay muted></audio>
    <!-- Mostrar el audio remoto -->
    <audio ref="remoteAudio" autoplay></audio>
  </div>
</template>

<script>
import io from 'socket.io-client';
import Peer from 'simple-peer';

export default {
  data () {
    return {
      socket: null,
      peer: null,
      myId: '', // ID único generado por el servidor
      targetId: '', // ID del destinatario
      localStream: null, // El flujo local (audio del micrófono)
      remoteStream: null, // Flujo remoto
    };
  },
  
  mounted () {
    this.initSocket();
  },

  methods: {
    // Inicializa la conexión Socket.io
    initSocket () {
      this.socket = io('http://localhost:3000'); // Conéctate al servidor

      // Recibir ID del socket
      this.socket.on('connect', () => {
        this.myId = this.socket.id;
        console.log('Connected with ID:', this.myId);
      });

      // Recibir señal
      this.socket.on('receive-signal', async (data) => {
        const { signal, from } = data;
        console.log('Signal received from:', from);
        await this.initialDevices();
        if (!this.peer) {
          // Crear Peer si no existe (modo receptor)
          this.createPeer(false, from);
        }
        this.peer.signal(signal);
        console.log('signal', { signal }); // Pasar la señal al Peer
      });
    },

    async initialDevices() {
      try {
        const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
        this.localStream = stream;
        // Asignar el flujo al elemento de audio local
        this.$refs.localAudio.srcObject = stream;
        console.log('TU ADIO SE CAPTURÓ CORRECTAMENTE')
      } catch (error) {
        console.error('Error getting local stream', error);
      }
    },

    // Crear la conexión Peer
    createPeer (isInitiator, to) {
      // Crear instancia de Peer
      this.peer = new Peer({
        initiator: isInitiator, // Determina si este Peer inicia la conexión
        trickle: false, // Evita múltiples señales pequeñas
        stream: this.localStream, // Pasar el flujo local (audio)
      });

      // Escuchar eventos del Peer
      this.peer.on('signal', (signal) => {
        console.log('Generated signal:', signal);
        this.socket.emit('send-signal', { signal, to });
      });

      this.peer.on('stream', (remoteStream) => {
        console.log('Received remote stream');
        this.remoteStream = remoteStream;
        // Asignar el flujo remoto al elemento de audio remoto
        this.$refs.remoteAudio.srcObject = this.remoteStream;
      });

      this.peer.on('connect', () => {
        console.log('Peer connected!');
      });


      this.peer.on('data', (data) => {
        console.log('Received data:', data.toString());
      });

      this.peer.on('close', () => {
        console.log('Peer connection closed');
        this.peer = null;
        this.endCall();
      });

      this.peer.on('error', (err) => {
        console.error('Peer error:', err);
      });
    },

    // Iniciar llamada
    async startCall () {
      try {
        const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
        this.localStream = stream;
        // Asignar el flujo al elemento de audio local
        this.$refs.localAudio.srcObject = stream;
        console.log('TU AUDIO SE CAPTURÓ CORRECTAMENTE')
        console.log('Starting call to:', this.targetId);
        this.createPeer(true, this.targetId); // Inicia la llamada
      } catch (error) {
        console.error('Error getting local stream', error);
      }
    },

    // Terminar llamada
    endCall () {
      this.localStream = null;
      this.remoteStream = null;
      this.$refs.localAudio.srcObject = null;
      this.$refs.remoteAudio.srcObject = null;

      if (this.peer) {
        this.peer.destroy();
        this.peer = null;
        console.log('Call ended');
      }
    },
  },
};
</script>
