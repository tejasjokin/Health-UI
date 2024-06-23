<template>
  <div>
    <div v-if="serverData">
      <p>Doctor ID: {{ serverData.doctor_id }}</p>
      <p>Date: {{ serverData.date }}</p>
      <div v-if="!serverData.consent">
        <select v-model="consentChoice" class="consent-select">
          <option value="">Select your consent</option>
          <option value="yes">Yes</option>
          <option value="no">No</option>
        </select>
        <button @click="submitConsent" :disabled="consentChoice === ''">Submit Consent</button>
      </div>
      <div v-else class="received-message">
        Consent: {{ serverData.consent }}
      </div>
    </div>
    <div v-else>
      <p>Loading server data...</p>
    </div>

    <!-- Modal for Private Key Input -->
    <div class="modal" v-if="showPrivateKeyModal">
      <div class="modal-content">
        <h3>Enter Private Key</h3>
        <input type="password" v-model="privateKey" placeholder="Enter your private key">
        <button @click="submitPrivateKey">Submit</button>
      </div>
    </div>
  </div>
</template>

<script>
import io from 'socket.io-client';
import CryptoJS from 'crypto-js';
import * as EC from 'elliptic';

const ec = new EC.ec('secp256k1'); // Specify the elliptic curve (secp256k1 for Bitcoin)

export default {
  data() {
    return {
      serverData: null,
      consentChoice: '',
      showPrivateKeyModal: false,
      privateKey: '',
    };
  },
  created() {
    this.socket = io('http://localhost:5000');

    // Check if serverData is stored in localStorage
    const storedData = localStorage.getItem('serverData');
    if (storedData) {
      this.serverData = JSON.parse(storedData);
    }

    // Listen for 'patient_details' event
    this.socket.on('patient_details', (data) => {
      if (data) {
        this.serverData = data;
        localStorage.setItem('serverData', JSON.stringify(data));
      } else {
        console.error('No server data received');
      }
    });

    // Listen for 'consent received' event
    this.socket.on('consent received', (data) => {
      if (this.serverData) {
        this.serverData.consent = data.message; // Assuming 'message' contains consent status
        localStorage.setItem('serverData', JSON.stringify(this.serverData));
      }
    });

    // Request server data when component is created
    this.socket.emit('request server data');
  },
  methods: {
    submitConsent() {
      if (this.consentChoice === '') {
        return; // Don't send if consent choice is not selected
      }
      // Show modal to ask for private key
      this.showPrivateKeyModal = true;
    },
    submitPrivateKey() {
      // Generate key pair from the private key entered by user
      const key = ec.keyFromPrivate(this.privateKey, 'hex');

      // Hash the consent choice (for demonstration purposes, you can adjust this based on your actual hashing needs)
      const hashedConsent = CryptoJS.SHA256(this.consentChoice).toString(CryptoJS.enc.Hex);

      // Sign the hashed consent with ECDSA
      const signature = key.sign(hashedConsent, 'base64').toDER('hex');

      // Send encrypted consent to Flask-SocketIO server along with any other required data
      this.socket.emit('send consent', {
        Consent: this.consentChoice, // Send plain consent
        Signature: signature, // Send ECDSA signature
        DoctorID: this.serverData.doctor_id, // Send doctor ID from serverData
        // Include any other data you need to send to the server
      });
      
      // Close the private key modal after consent is sent
      this.showPrivateKeyModal = false;
    }
  },
  beforeUnmount() {
    if (this.socket) {
      this.socket.disconnect();
    }
  }
};
</script>

<style scoped>
.select-container {
  display: flex;
  justify-content: center;
  align-items: center;
  height: 100%;
}

.consent-select {
  width: 150px; /* Adjust width as needed */
  margin-right: 10px;
}

.received-message {
  margin-top: 10px;
  text-align: center;
}

.modal {
  display: flex;
  justify-content: center;
  align-items: center;
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.5);
}

.modal-content {
  background-color: white;
  padding: 20px;
  border-radius: 5px;
  text-align: center;
}
</style>
