<template>
  <div class="schedule">
    <table class="schedule__table"
           @click="checkEvent($event)"
           @mousedown="mouseButtonDown = !mouseButtonDown"
           @mouseup="mouseButtonDown = !mouseButtonDown"
           @mouseover="highlightCells($event)"
    >
      <tr class="schedule__row schedule__head">
        <td></td>
        <td>
          <span class="schedule__subtitle">all day</span>
        </td>
        <td colspan="3"
            v-for="item of timeMarkers"
            :key="item">
        <span class="schedule__marker">
          {{ item | convertTimeMarkers }}
        </span>
        </td>
      </tr>
      <tr class="schedule__row"
          v-for="(day, index) of daysOfWeek"
          :key="index">
        <td :class="[data[day].length > 0 ? 'filled' : '']">
          <span class="schedule__subtitle">{{ day }}</span>
        </td>
        <td :class="['schedule__all-day', checkIntervalsLength(day) ? 'full' : '']"
            :data-day="day">
        </td>
        <td v-for="hour of 24"
            :key="hour"
            :class="['schedule__period' ,getIntervals(day, hour) ? 'reserved' : '']"
            :data-day="day"
            :data-index="hour"
        >
        </td>
      </tr>
    </table>
    <div class="schedule__btn-wrapper">
      <button type="button"
              class="schedule__btn"
              @click="clearSchedule"
      >Clear</button>
      <button type="button"
              class="schedule__btn"
              @click="convertToJson"
      >Save Changes</button>
    </div>
  </div>
</template>

<script>
import json from '@/json/data.json';

export default {
  name: "Schedule",
  data () {
    return {
      data: json,
      daysOfWeek: ["mo","tu","we","th","fr","sa","su"],
      timeMarkers: [0,3,6,9,12,15,18,21],
      mouseButtonDown: false,
    }
  },
  methods: {
    getIntervals (day, time) {
      const intervals = this.data[day];
      if (intervals.length > 0) {
        return intervals.some((interval) => time >= ((interval.bt) / 60) + 1 && time <= ((interval.et + 1) / 60));
      }
    },
    checkIntervalsLength (day) {
      const intervalLength = this.data[day].reduce((accum, value) => {return (accum + (value.et - value.bt))}, 0);
      return intervalLength === 1439;
    },
    checkEvent (e) {
      if (e.target.closest('td.schedule__all-day')) {
        this.fillInOrDeleteAllDay(e);
      } else if (e.target.closest('td.schedule__period')) {
        this.checkIfItBelongsToTheInterval(e);
      }
    },
    fillInOrDeleteAllDay (e) {
      const day = e.target.dataset.day;
      if (e.target.classList.contains('full')) {
        this.data[day] = [];
      } else this.data[day] = [{
        bt: 0,
        et: 1439
      }];
    },
    checkIfItBelongsToTheInterval (e) {
      const day = e.target.dataset.day;
      const hourIndex = parseInt(e.target.dataset.index);
      let intervalIndex = null;
      const detected = this.data[day].find((item, index) => {
        intervalIndex = index;
        return (item.bt <= (hourIndex - 1) * 60 && hourIndex * 60 <= item.et + 1)
      });
      if (detected) {
        //actions if the segment is included in the interval
        e.target.classList.remove('reserved');
        if (this.data[day][intervalIndex].bt === (hourIndex - 1) * 60 && hourIndex * 60 === this.data[day][intervalIndex].et + 1) {
          this.data[day].splice(intervalIndex, 1);
        } else if (this.data[day][intervalIndex].bt === (hourIndex - 1) * 60) {
          this.data[day][intervalIndex].bt += 60;
        } else if (this.data[day][intervalIndex].bt < (hourIndex - 1) * 60 && hourIndex * 60 < this.data[day][intervalIndex].et + 1) {
          this.data[day].splice(intervalIndex, 1,
              {bt: this.data[day][intervalIndex].bt, et: (hourIndex - 1) * 60 - 1},
              {bt: hourIndex * 60, et: this.data[day][intervalIndex].et}
          );
        } else if (hourIndex * 60 === this.data[day][intervalIndex].et + 1) {
          this.data[day][intervalIndex].et = (this.data[day][intervalIndex].et - 60);
        }
      } else {
        //actions if the segment is not included in the interval
        let rightIndex = null;
        let leftIndex = null;
        const right = this.data[day].find((item, index) => {
          rightIndex = index;
          return (item.et === (hourIndex - 1) * 60 - 1);
        });
        const left = this.data[day].find((item, index) => {
          leftIndex = index;
          return (item.bt === hourIndex * 60);
        });
        if (left && right) {
          const obj = {
            bt: this.data[day][rightIndex].bt,
            et: this.data[day][leftIndex].et,
          }
          this.data[day].splice(rightIndex, 1);
          this.data[day].splice(leftIndex - 1, 1);
          this.data[day].push(obj);
        } else if (right) {
          this.data[day][intervalIndex].et += 60;
        } else if (left) {
          this.data[day][intervalIndex].bt -= 60;
        }  else {
          e.target.classList.add('reserved');
          this.data[day].push({
            bt: (hourIndex - 1) * 60,
            et: hourIndex * 60 - 1,
          });
        }
      }
    },
    highlightCells (e) {
      const elem = e.target.closest('td.schedule__period');
      if (this.mouseButtonDown && elem) {
        if (!elem.classList.contains('reserved')) {
          this.checkIfItBelongsToTheInterval(e)
        }
      }
    },
    clearSchedule () {
      for (const day in this.data) {
        if (Object.prototype.hasOwnProperty.call(this.data, day)) {
          this.data[day] = [];
        }
      }
    },
    convertToJson () {
      const json = JSON.stringify(this.data);
      console.log(json)
    }
  },
  filters: {
    convertTimeMarkers (marker) {
      return (marker < 10) ? `0${marker}.00` : `${marker}.00`;
    }
  }
}
</script>

<style lang="scss">
@import "../assets/main";
</style>