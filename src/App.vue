<template>
  <div class="w-full h-full flex flex-col justify-end relative">
    <div
      class="absolute top-0 left-0 bg-gray-900 text-white py-2 px-4 font-semibold"
    >
      {{ (currentTime / 1000) | parseLocalTime }}
    </div>
    <div class="absolute top-0 right-0 mt-8 mr-10">
      <img class="h-28 w-auto" src="@/assets/spoutcam.png" />
    </div>
    <div
      class="flex flex-row justify-between items-center bg-black px-4 py-2 text-white"
    >
      <div class="flex flex-row space-x-2">
        <img
          class="h-28 w-auto mx-6"
          v-if="currentConditions.icon !== null"
          :src="
            'https://s3.amazonaws.com/tempest.cdn/assets/better-forecast/v9/' +
              currentConditions.icon +
              '.svg'
          "
        />
        <div class="flex flex-col justify-center space-y-2">
          <h1 class="text-5xl text-shadow">
            {{ temperature | parseCAsF }}&deg;F
          </h1>
          <h1 class="text-3xl text-shadow tracking-wide">
            {{ currentConditions.text }}
          </h1>
        </div>
        <div
          class="flex flex-row text-white px-20 space-x-10 mt-16 mx-auto text-lg"
        >
          <div class="flex flex-row items-center">
            <img
              class="h-8 w-8 filter-white mr-2"
              src="https://tempestwx.com/images/sunrise.svg"
            />
            {{ dayInfo.sunrise | parseLocalTime }}
          </div>
          <div class="flex flex-row items-center">
            <img
              class="h-8 w-8 filter-white mr-2"
              src="https://tempestwx.com/images/sunset.svg"
            />
            {{ dayInfo.sunset | parseLocalTime }}
          </div>
          <div class="flex flex-row items-center">
            <img
              class="h-5 w-5 filter-white mr-2"
              src="https://tempestwx.com/images/cc-humidity.svg"
            />
            {{ humidity }}% humidity
          </div>
          <div class="flex flex-row items-center">
            <img
              id="activeIcon"
              class="h-5 w-5 filter-white mr-2"
              src="https://tempestwx.com/images/cc-pressure-trend-arrow.svg"
              :data-pressure-trend="pressureTrend"
            />
            {{ pressure }} mb
          </div>
          <div class="flex flex-row items-center">
            <img
              class="h-5 w-5 filter-white mr-2"
              src="https://s3.amazonaws.com/tempest.cdn/assets/better-forecast/v9/chance-rain.svg"
            />
            {{ rainAccumulated | mmToIn }}"
          </div>
          <div class="flex flex-row items-center">
            <img
              class="h-6 w-6 filter-white mr-2"
              src="https://tempestwx.com/images/cc-uv.svg?v=20210614b"
            />
            {{ uv | round(1) }} UV
          </div>
        </div>
      </div>
      <div class="overflow-hidden w-36 h-36 flex relative mr-2">
        <div
          class="w-full h-full text-white text-xl felx-grow flex flex-col items-center content-center justify-center"
        >
          <h1 class="text-5xl text-shadow">
            {{ convertMsToMph(wind.speed) | round(10) }}
          </h1>
          <span class="text-sm text-shadow">MPH</span>
          <!-- <span class="text-sm">{{
            wind.speed !== 0 ? degreesToDirection(wind.direction) : "---"
          }}</span> -->
        </div>
        <div class="weather-tile-icon absolute top-0 left-0 w-36 h-36">
          <div
            class="wind-direction-indicator w-full h-full"
            :style="{
              transform: 'rotate(' + wind.direction + 'deg)',
            }"
            v-if="wind.speed !== 0 && wind.speed !== null"
          ></div>
        </div>
      </div>
    </div>
    <!-- <div class="w-full flex flex-row bg-yellow-500 p-2">
      Special Marine Warning
    </div> -->
  </div>
</template>

<script>
import axios from "axios";
import moment from "moment";

export default {
  name: "App",
  data() {
    return {
      apiKey: process.env.VUE_APP_API_KEY,
      deviceIds: [45477, 46971], // mine 72423
      stationId: 12997,
      socket: null,
      validTs: null,
      wind: {
        speed: null,
        direction: null,
        avgDirection: null,
        gust: null,
        average: null,
      },
      lightning: {
        count: null,
        avgDistance: null,
        lastStrikeTs: null,
        lastDistance: null,
        lastEnergy: null,
      },
      temperature: null,
      pressure: null,
      pressureTrend: "rising",
      humidity: null,
      uv: null,
      illuminance: null,
      rainAccumulated: null,
      solarRadiation: null,
      currentConditions: {
        icon: null,
        text: null,
      },
      dayInfo: {
        sunrise: null,
        sunset: null,
      },
      currentConditionsInterval: null,
      currentTime: null,
      currentTimeInterval: null,
    };
  },
  mounted() {
    this.openWebSocketConnection();
    this.updateCurrentConditions();
    this.currentConditionsInterval = setInterval(
      this.updateCurrentConditions,
      5 * 60 * 1000
    );
    this.currentTimeInterval = setInterval(this.updateCurrentTime, 1000);
  },

  destroyed() {
    if (this.socket !== null) {
      this.socket.close();
      this.socket = null;
    }
    if (this.currentConditionsInterval !== null) {
      clearInterval(this.currentConditionsInterval);
      this.currentConditionsInterval = null;
    }
    if (this.currentTimeInterval !== null) {
      clearInterval(this.currentTimeInterval);
      this.currentTimeInterval = null;
    }
  },
  methods: {
    openWebSocketConnection() {
      this.socket = new WebSocket(
        "wss://ws.weatherflow.com/swd/data?token=" + this.apiKey
      );
      this.socket.onopen = () => {
        this.deviceIds.forEach((dId) => {
          // request observation updates
          var listen = {
            type: "listen_start",
            device_id: dId,
            id: dId + "-listen",
          };
          this.socket.send(JSON.stringify(listen));

          // request rapid wind updates
          var rapid = {
            type: "listen_rapid_start",
            device_id: dId,
            id: dId + "-rapid",
          };
          this.socket.send(JSON.stringify(rapid));
        });
      };
      this.socket.onmessage = this.onWebSocketMessage;
      this.socket.onerror = this.onWebSocketError;
      this.socket.onclose = this.onWebSocketClose;
    },
    onWebSocketMessage(event) {
      var message = JSON.parse(event.data);

      if (message["type"] == "rapid_wind") {
        // 0	Time Epoch	Seconds
        // 1	Wind Speed	m/s
        // 2	Wind Direction	Degrees

        this.validTs = message["ob"][0];
        this.wind.speed = message["ob"][1];
        this.wind.direction = message["ob"][2];
      }

      if (message["type"] == "obs_air") {
        // 0	Time Epoch	Seconds
        // 1	Station Pressure	MB
        // 2	Air Temperature	C
        // 3	Relative Humidity	%
        // 4	Lightning Strike Count
        // 5	Lightning Strike Avg Distance	km
        // 6	Battery	Volts
        // 7	Report Interval	Minutes

        let ob = message["obs"][0];
        this.validTs = ob[0];
        this.pressure = ob[1];
        this.temperature = ob[2];
        this.humidity = ob[3];
        this.lightning.count = ob[4];
        this.lightning.avgDistance = ob[5];
      }

      if (message["type"] == "obs_sky") {
        // 0	Time Epoch	Seconds
        // 1	Illuminance	Lux
        // 2	UV	Index
        // 3	Rain Accumulated	mm
        // 4	Wind Lull (minimum 3 second sample)	m/s
        // 5	Wind Avg (average over report interval)	m/s
        // 6	Wind Gust (maximum 3 second sample)	m/s
        // 7	Wind Direction	Degrees
        // 8	Battery	Volts
        // 9	Report Interval	Minutes
        // 10	Solar Radiation	W/m^2
        // 11	Local Daily Rain Accumulation	mm
        // 12	Precipitation Type	0 = none, 1 = rain, 2 = hail
        // 13	Wind Sample Interval	seconds
        // 14	Rain Accumulated Final (Rain Check)	mm
        // 15	Local Daily Rain Accumulation Final (Rain Check)	mm
        // 16	Precipitation Analysis Type
        // 16-0 = none
        // 16-1 = Rain Check with user display on
        // 16-2 = Rain Check with user display off

        let ob = message["obs"][0];
        this.validTs = ob[0];
        this.illuminance = ob[1];
        this.uv = ob[2];

        this.wind.average = ob[5];
        this.wind.gust = ob[6];
        this.wind.avgDirection = ob[7];
        this.rainAccumulated = ob[11];
      }

      if (message["type"] == "obs_st") {
        // 0	Time Epoch	Seconds
        // 1	Wind Lull (minimum 3 second sample)	m/s
        // 2	Wind Avg (average over report interval)	m/s
        // 3	Wind Gust (maximum 3 second sample)	m/s
        // 4	Wind Direction	Degrees
        // 5	Wind Sample Interval	seconds
        // 6	Station Pressure	MB
        // 7	Air Temperature	C
        // 8	Relative Humidity	%
        // 9	Illuminance	Lux
        // 10	UV	Index
        // 11	Solar Radiation	W/m^2
        // 12	Rain Accumulated	mm
        // 13	Precipitation Type	0 = none, 1 = rain, 2 = hail
        // 14	Lightning Strike Avg Distance	km
        // 15	Lightning Strike Count
        // 16	Battery	Volts
        // 17	Report Interval	Minutes
        // 18	Local Daily Rain Accumulation	mm
        // 19	Rain Accumulated Final (Rain Check)	mm
        // 20	Local Daily Rain Accumulation Final (Rain Check)	mm
        // 21	Precipitation Analysis Type
        // 21-0 = none
        // 21-1 = Rain Check with user display on
        // 21-2 = Rain Check with user display off

        let ob = message["obs"][0];
        this.validTs = ob[0];

        this.wind.average = ob[2];
        this.wind.gust = ob[3];
        this.wind.avgDirection = ob[4];

        this.pressure = ob[6];
        this.temperature = ob[7];
        this.humidity = ob[8];
        this.illuminance = ob[9];
        this.uv = ob[10];
        this.solarRadiation = ob[11];
        this.lightning.count = ob[15];
        this.lightning.avgDistance = ob[14];

        this.rainAccumulated = ob[18];
        this.pressureTrend = message.summary.pressure_trend;
      }

      if (message["type"] == "evt_strike") {
        this.lightning.lastStrikeTs = message["evt"][0];
        this.lightning.lastDistance = message["evt"][1];
        this.lightning.lastEnergy = message["evt"][2];
      }
    },
    onWebSocketError(event) {
      console.log(event);
    },
    onWebSocketClose(event) {
      console.log(event);
    },
    async updateCurrentConditions() {
      console.log("updating current conditions");
      let res = await axios.get(
        "https://swd.weatherflow.com/swd/rest/better_forecast?api_key=" +
          this.apiKey +
          "&station_id=" +
          this.stationId
      );
      let currentConditions = res.data.current_conditions;
      let todayForecastInfo = res.data.forecast.daily[0];
      this.currentConditions = {
        icon: currentConditions.icon,
        text: currentConditions.conditions,
      };
      this.dayInfo = {
        sunrise: todayForecastInfo.sunrise,
        sunset: todayForecastInfo.sunset,
      };
    },
    convertMsToMph(val) {
      return val * 2.23694;
    },
    degreesToDirection(degrees) {
      if (degrees >= 348.75 && degrees < 11.25) return "N";
      if (degrees >= 11.25 && degrees < 33.75) return "NNE";
      if (degrees >= 33.75 && degrees < 56.25) return "NE";
      if (degrees >= 56.25 && degrees < 78.75) return "ENE";
      if (degrees >= 78.75 && degrees < 101.25) return "E";
      if (degrees >= 101.25 && degrees < 123.75) return "SE";
      if (degrees >= 123.75 && degrees < 146.25) return "SSE";
      if (degrees >= 146.25 && degrees < 168.75) return "S";
      if (degrees >= 191.25 && degrees < 213.75) return "SSW";
      if (degrees >= 213.75 && degrees < 236.25) return "SW";
      if (degrees >= 236.25 && degrees < 258.75) return "WSW";
      if (degrees >= 258.75 && degrees < 281.25) return "W";
      if (degrees >= 281.25 && degrees < 303.75) return "WNW";
      if (degrees >= 303.75 && degrees < 326.25) return "NW";
      if (degrees >= 326.25 && degrees < 348.75) return "NNW";
    },
    updateCurrentTime() {
      this.currentTime = Date.now();
    },
  },
  filters: {
    round(num, place) {
      return Math.round(num * place) / place;
    },

    parseCAsF(data) {
      return Math.round((data * (9 / 5) + 32) * 10) / 10;
    },
    parseLocalTime(time) {
      return moment.unix(time).format("h:mm A");
    },
    mmToIn(mm) {
      if (mm == null || mm == 0) return "0.00";
      return Math.round((mm / 25.4) * 100) / 100;
    },
  },
};
</script>

<style>
body {
  /* background-color: #000; */
  margin: 0;
  padding: 0;
}
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  text-align: center;
  color: #fff;
}
.weather-tile-icon {
  background-image: url("https://tempestwx.com/images/wind-deg-marks.svg?v=20210614b");
  filter: invert(100%) sepia(100%) saturate(0) hue-rotate(201deg)
    brightness(107%) contrast(101%);
  background-position: 50% 100%;
  background-repeat: no-repeat;
  color: white;
}
.wind-direction-indicator {
  background: transparent
    url("https://tempestwx.com/images/wind-sliver.svg?v=20210614b") no-repeat
    50% 3px;
  transition: all 1s ease;
}
.text-shadow {
  text-shadow: 1px 1px #000;
}
.filter-white {
  filter: invert(100%) sepia(100%) saturate(0) hue-rotate(201deg)
    brightness(107%) contrast(101%);
}
#activeIcon[data-pressure-trend="rising"] {
  transform: rotate(-45deg);
  -webkit-transform: rotate(-45deg);
  -ms-transform: rotate(-45deg);
}
#activeIcon[data-pressure-trend="falling"] {
  transform: rotate(45deg);
  -webkit-transform: rotate(45deg);
  -ms-transform: rotate(45deg);
}
</style>
