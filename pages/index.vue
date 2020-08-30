<template lang="pug">
div
  button(@click="requestAndConnect", :disabled="device") connect

  .color-sensor(:style="{ backgroundColor: `rgb(${sensors.rgb})` }")

  pre {{ sensors.motion }}
</template>

<script>
export default {
  data() {
    return {
      device: null,
      server: null,
      service: null,
      characteristic: null,
      sensors: {
        rgb: [],
        pants: [],
        motion: [],
        // last sticker?
        // last event?
      },
      state: { log: [], coins: 0 },
    };
  },
  computed: {},
  methods: {
    async requestAndConnect() {
      try {
        this.device = await navigator.bluetooth.requestDevice({
          filters: [
            {
              namePrefix: "LEGO Mario",
            },
          ],
          // acceptAllDevices: true,
          optionalServices: ["00001623-1212-efde-1623-785feabcd123"],
        });
      } catch (err) {
        return;
      }

      this.device.addEventListener(
        "gattserverdisconnected",
        this.handleDeviceDisconnect
      );

      this.server = await this.device.gatt.connect();

      this.service = await this.server.getPrimaryService(
        "00001623-1212-efde-1623-785feabcd123"
      );

      this.characteristic = await this.service.getCharacteristic(
        "00001624-1212-efde-1623-785feabcd123"
      );

      this.characteristic.addEventListener(
        "characteristicvaluechanged",
        this.handleValueEvent
      );

      this.characteristic.startNotifications();

      await this.subscribeToEvents();
      await this.subscribeToColor();
      // await this.subscribeToPants();
      // await this.subscribeToMotion();
    },
    async subscribeToEvents() {
      await this.characteristic.writeValue(
        new Uint8Array([
          10, // length of message (10)
          0, // hub id (Always 0)
          65, // Message type (Port Input Format Setup (Single))
          3, // port id
          2, // port mode
          5, // interval
          0, //
          0, //
          0, //
          1, //
        ])
      );
    },
    async subscribeToMotion() {
      await this.characteristic.writeValue(
        new Uint8Array([10, 0, 65, 0, 0, 5, 0, 0, 0, 1])
      );
    },
    async subscribeToPants() {
      await this.characteristic.writeValue(
        new Uint8Array([10, 0, 65, 2, 0, 5, 0, 0, 0, 1])
      );
    },
    async subscribeToColor() {
      await this.characteristic.writeValue(
        new Uint8Array([10, 0, 65, 1, 1, 5, 0, 0, 0, 1])
      );
    },
    handleDeviceDisconnect(event) {
      this.device = null;
      this.server = null;
      this.service = null;
      this.characteristic = null;
    },
    handleValueEvent(event) {
      const message = Array.from(new Uint8Array(event.target.value.buffer));

      const messageHeader = message.slice(0, 3);
      const messageBody = message.slice(3);

      const messageType = messageHeader[2];

      // console.log(messageHeader, messageBody);

      // Only handle incoming port values from here.
      // https://lego.github.io/lego-ble-wireless-protocol-docs/index.html#port-value-single
      if (messageType != 69) return;

      const port = messageBody[0];

      if (port === 1) {
        this.sensors.rgb = messageBody.slice(1);
      } else if (port === 0) {
        this.sensors.motion = messageBody.slice(1);
      } else if (port === 3) {
        // parse events
        console.log(message);
      }
    },
  },
};
</script>


<style scoped>
.color-sensor {
  width: 250px;
  height: 250px;
  margin: 20px;
  border-radius: 20px;
  border: 2px dashed #262626;
}

.motion-sensor {
  width: 250px;
  height: 250px;
  margin: 20px;
  border-radius: 20px;
  background-color: #cccccc;
  /* border: 2px dashed #262626; */
}
</style>