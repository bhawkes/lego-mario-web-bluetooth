<template lang="pug">
div
  button(@click="requestAndConnect") connect
</template>

<script>
export default {
  data() {
    return {
      device: null,
      server: null,
      service: null,
      characteristic: null,
    };
  },
  computed: {},
  methods: {
    async requestAndConnect() {
      this.device = await navigator.bluetooth.requestDevice({
        filters: [
          {
            namePrefix: "LEGO Mario",
          },
        ],
        // acceptAllDevices: true,
        optionalServices: ["00001623-1212-efde-1623-785feabcd123"],
      });

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

      this.subscribeToEvents();
    },
    async subscribeToEvents() {
      await this.characteristic.writeValue(
        new Uint8Array([
          0x0a, // length of message (10)
          0x00, // hub id (Always 0)
          0x41, // message type (Port Value (Single))

          0x03, // port id
          0x03, // mode

          0x05, // interval
          0x00, // notification?
          0x00, // ??
          0x00, // ??
          0x01, // ??
        ])
      );
    },
    handleValueEvent(event) {
      console.log(new Uint8Array(event.target.value.buffer));
    },
  },
};
</script>
