<template>
  <div>
    <h1>Simple Peer - Audio Call</h1>
    <p>Your ID: {{ myId }}</p>
    <input v-model="targetId" placeholder="Enter Target ID" />
    <button @click="startCall">Start Call</button>
    <button @click="endCall">End Call</button>

    <!-- Mostrar el audio local-->
    <audio ref="localAudio" autoplay muted></audio>
    <!-- Mostrar el audio remoto -->
    <audio ref="remoteAudio" autoplay></audio>

    <!-- Opciones de respuesta a la llamada (solo para el receptor) -->
    <div v-if="incomingCall">
      <p>Incoming Call from {{ callFromId }}</p>
      <button @click="acceptCall">Accept</button>
      <button @click="rejectCall">Reject</button>
    </div>
  </div>
</template>

<script>
import io from 'socket.io-client';
import Peer from 'simple-peer';

export default {
  data() {
    return {
      socket: null,
      peer: null,
      myId: '', // ID único generado por el servidor
      targetId: '', // ID del destinatario
      localStream: null, // El flujo local (audio del micrófono)
      incomingCall: false, // Para saber si hay una llamada entrante
      callFromId: '', // ID del usuario que llama
      remoteStream: null, // Flujo remoto
    };
  },
  mounted() {
    this.initSocket();
  },
  methods: {
    // Inicializa la conexión Socket.io
    initSocket() {
      this.socket = io('http://localhost:3000'); // Conéctate al servidor

      // Recibir ID del socket
      this.socket.on('connect', () => {
        this.myId = this.socket.id;
        console.log('Connected with ID:', this.myId);
      });

      // Recibir señal de llamada
      this.socket.on('receive-signal', async (data) => {
        const { signal, from } = data;
        console.log('Signal received from:', from);
        await this.initialDevices();
        if (!this.peer) {
          // Mostrar opciones de llamada entrante
          this.incomingCall = true;
          this.callFromId = from; // Guardar el ID de quien llama
          this.createPeer(false, from);
          console.log("Received signal, passing to peer:", signal);
        }
        this.peer.signal(signal); // Pasar la señal al Peer
      });
    },

    // Crear el flujo de audio local
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
    createPeer(isInitiator, to) {
      console.log(`Creating peer: ${isInitiator ? 'Initiator' : 'Receiver'}`);

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

      this.peer.on('connect', () => {
        console.log('Peer connected!');
      });

      // Cuando recibimos el flujo remoto, lo asignamos al audio remoto
      /*this.peer.on('stream', (remoteStream) => {
        console.log('Received remote stream', remoteStream);
        if (this.$refs.remoteAudio) {
          console.log('Assigning remote stream to audio element');
          this.$refs.remoteAudio.srcObject = remoteStream; // Asignar el flujo remoto al elemento de audio remoto
        }
      });*/

      this.peer.on('stream', (remoteStream) => {
        console.log('Received remote stream');
        this.remoteStream = remoteStream;
        // Asignar el flujo remoto al elemento de audio remoto
        this.$refs.remoteAudio.srcObject = remoteStream;
      });

      this.peer.on('data', (data) => {
        console.log('Received data:', data.toString());
      });

      this.peer.on('close', () => {
        console.log('Peer connection closed');
        this.peer = null;
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

    // Aceptar llamada entrante
    acceptCall() {
      console.log('Call accepted from:', this.callFromId);
      this.createPeer(false, this.callFromId); // Crear peer como receptor
      this.incomingCall = false; // Ocultar la interfaz de llamada entrante
    },

    // Rechazar llamada entrante
    rejectCall() {
      console.log('Call rejected from:', this.callFromId);
      this.incomingCall = false; // Ocultar la interfaz de llamada entrante
      this.peer = null; // No crear ninguna conexión
    },

    // Terminar llamada
    endCall() {
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

<style scoped>
/* Agregar estilos básicos para los botones y el layout */
button {
  margin: 10px;
  padding: 10px;
  font-size: 16px;
}
input {
  padding: 10px;
  font-size: 16px;
  margin-bottom: 10px;
}
</style>
