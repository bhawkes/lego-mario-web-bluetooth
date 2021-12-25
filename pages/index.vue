<template lang="pug">
div
  .container.mb-5
    .row
      .col
        h1 Lego Mario
      .col.align-self-center
        button(@click="requestAndConnect", :disabled="device") connect

  .container
    .row
      .col
        h2.mb-4 Sensors
        //- .sensor.mb-2
          .row
            .col-4
              .sensor__box.sensor__box--tag
            .col-8
              h3.mb-3 Tag
        .sensor.mb-2
          .row
            .col-4
              .sensor__box.sensor__box--color(
                :style="{ backgroundColor: `rgb(${sensors.rgb})` }"
              )
            .col-8
              h3.mb-3 RGB
              pre r:{{ sensors.rgb[0] }}, g:{{ sensors.rgb[1] }}, b:{{ sensors.rgb[2] }}
        .sensor.mb-2
          .row
            .col-4
              .sensor__box.sensor__box--pants
            .col-8
              h3.mb-3 Pants
              pre {{ JSON.stringify(sensors.pants) }}
        .sensor.mb-2
          .row
            .col-4
              .sensor__box.sensor__box--motion
            .col-8
              h3.mb-3 Motion
              pre {{ JSON.stringify(sensors.motion) }}
      .col
        h2.mb-4 Score = {{total}}
        h3(v-for="entry in scores", v-if="entry.value") {{ events.types[entry.type] ? events.types[entry.type].name : '???' }} = {{ entry.value }}
      .col
        h2.mb-4 Events
        table.u--w_100
          thead
            tr
              th id
              th name
              th key
              th value
          tbody
            tr(v-for="entry in state.log")
              td {{ entry.type }}
              td {{ events.types[entry.type] ? events.types[entry.type].name : '???' }}
              td {{ events.keys[entry.key] ? events.keys[entry.key] : entry.key }}
              td {{ entry.value }}
</template>

<script>
import stickers from "~/assets/stickers.json";
import events from "~/assets/events.json";

export default {
  data() {
    return {
      device: null,
      server: null,
      service: null,
      characteristic: null,

      stickers,
      events,
      sensors: {
        rgb: [0, 0, 0],
        pants: [],
        motion: []
        // last sticker?
        // last event?
      },
      state: {
        log: [],
        coins: 0,
        totals: events.types
      }
    };
  },
  computed: {
    scores() {
      let scores = {};

      this.state.log.forEach(entry => {
        if (entry.key === 32) {
          scores[entry.type] = entry;
        }
      });

      return scores;
    },
    total() {
      return Object.values(this.scores).reduce((accumulator, entry) => {
        return accumulator + entry.value;
      }, 0);
    }
  },
  methods: {
    async requestAndConnect() {
      try {
        this.device = await navigator.bluetooth.requestDevice({
          filters: [
            {
              namePrefix: "LEGO Mario"
            }
          ],
          // acceptAllDevices: true,
          optionalServices: ["00001623-1212-efde-1623-785feabcd123"]
        });
        console.log("Device connected", this.device)
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
      await this.subscribeToPants();
      await this.subscribeToMotion();
    },
    async subscribeToEvents() {
      await this.characteristic.writeValue(
        new Uint8Array([
          10, // length of message (10)
          0, // hub id (Always 0)
          65, // Message type (Port Input Format Setup (Single))
          3, // port id
          2, // port mode
          1, // interval
          0, //
          0, //
          0, //
          1 //
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

      if (messageType !== 69) return;

      const port = messageBody[0];

      if (port === 0) {
        this.sensors.motion = messageBody.slice(1);
        // console.log('Motion:', this.sensors.motion.toString())
      } else if (port === 1) {
        this.sensors.rgb = messageBody.slice(1);
        // console.log('RGB:', this.sensors.rgb.toString())
      } else if (port === 2) {
        this.sensors.pants = messageBody.slice(1);
        // console.log('Pants:', this.sensors.pants.toString())
      } else if (port === 3) {
        const eventType = messageBody[1];
        const eventKey = messageBody[2];
        const eventValue = messageBody[3] + messageBody[4] * 256;

        // console.log('Event:', eventType, eventKey, eventValue)
        this.parseEvent(eventType, eventKey, eventValue);
      } else {
        console.log(`Unhandled event: port=${port} messageType=${messageType} messageHeader=${messageHeader} messageBody=${messageBody}`)
      }
    },
    parseEvent(type, key, value) {
      this.state.log.push({
        type,
        key,
        value
      });

      if (key === 32) {
        this.state.totals[type].coins = value;
      }
    }
  }
};
</script>

<style scoped>
h1,
h2,
h3,
h4,
h5,
h6,
p,
pre {
  margin: 0;
}

.color-sensor {
  width: 100px;
  height: 100px;
  /* margin: 20px; */
  border-radius: 5px;
  border: 2px dashed #262626;
}

.sensor {
  /* display: flex;
  align-items: flex-start; */
}

.sensor__box {
  width: 100px;
  height: 100%;
  display: inline-block;
  padding-top: 100%;
  /* margin: 20px; */
  border-radius: 5px;
  border: 2px dashed #262626;
}

.grid {
}

.grid__row {
  display: flex;
  flex-wrap: wrap;
}

.grid__col {
  flex: 1;
}

.u--align-center {
  align-items: center;
}

.u--space-between {
  justify-content: space-between;
}

.u--w_100 {
  width: 100%;
}
</style>
