<script>
import { objectToString } from '@vue/shared';
import ky from 'ky';
export default {
  data() {
    return {
      userReviveData: {},
      userEventsData: {},
      userApi: '',
      factionSearchData: '',
      state: '',
    }
  },

  computed: {
    userReviveDataAsAnArray() {
      return Object.values(this.userReviveData)
    },

    usersRevivedAsObject() {
      return this.userReviveDataAsAnArray.reduce((accumulator, userReviveEntry) => {
        const localAccumulator = accumulator[userReviveEntry.target_id] !== undefined ? accumulator : {
          ...accumulator,
          [userReviveEntry.target_id]: {
            name: userReviveEntry.target_name,
            id: userReviveEntry.target_id,
            faction: userReviveEntry.target_factionname,
            reviveSuccess: 0,
            reviveFailure: 0,
          }
        };
        const target = localAccumulator[userReviveEntry.target_id]
        if (userReviveEntry.result === 'success') {
          return {
            ...localAccumulator,
            [userReviveEntry.target_id]: {
              ...target,
              reviveSuccess: target.reviveSuccess + 1
            }
          }
        }
        return {
          ...localAccumulator,
          [userReviveEntry.target_id]: {
            ...target,
            reviveFailure: target.reviveFailure + 1
          }
        };
      }, {})
    },

    usersRevivedAsAnArray() {
      return Object.values(this.usersRevivedAsObject).map(item => {
        return {
          ...item,
          reviveTotal: item.reviveSuccess + item.reviveFailure,
          events: this.parsedUserEventsAsAnArray.filter(log => log.id === item.id),
          faction: item.faction
        }
      })
    },

    usersEventsAsAnArray() {
      return Object.values(this.userEventsData).map(item => item.event)
    },

    parsedUserEventsAsAnArray() {
      const getIdRegex = /XID\=([0-9]+)(?:"\>|\>)(.+)\<.*/gm;
      const stripAnchorRegex = /(<\/?[a|A][^>]*>|)/gm;
      return this.usersEventsAsAnArray
        .map(item => ({
          match: getIdRegex.exec(item),
          message: item.replaceAll(stripAnchorRegex, "")
        }))
        .filter(item => item.match !== null)
        .map(item => {
          return {
            id: parseInt(item.match[1]),
            message: item.message
          }
        })
    },

    usersRevivedAsAnArrayFilteredByFaction() {
      const factionSearchParam = this.factionSearchData
      if (factionSearchParam === "") {
        return this.usersRevivedAsAnArray
      }
      return this.usersRevivedAsAnArray
        .filter(item => item.faction.toUpperCase() === factionSearchParam.toUpperCase())
    }

  },

  methods: {


    async getUserReviveData() {
      const { revives } = await ky.get(`https://api.torn.com/user/`, {
        searchParams:
        {
          key: this.userApi,
          selections: 'revives',
        }
      }).json();
      return revives;
    },

    async getUserEventData() {
      const { events } = await ky.get(`https://api.torn.com/user/`, {
        searchParams:
        {
          key: this.userApi,
          selections: 'events',
        }
      }).json();
      return events;
    },

    async condensedUserData() {
      try {
        this.state = 'loading';
        const [revives, events] = await Promise.all([this.getUserReviveData(), this.getUserEventData()])
        this.userReviveData = revives;
        this.userEventsData = events;
        this.state = 'success';
      } catch (err) {
        this.state = 'error';
        console.error("Get all data failed", err)
      }
    },

  }
}

</script>

<template>
  <!-- <pre>{{ usersRevivedAsAnArrayFilteredByFaction }}</pre> -->
  <div class="screen-size"></div>
  <header class="header">
    <h1>Torn Revive Ledger</h1>
  </header>
  <div class="search-grid">

    <div>
      <label for="userApi">Your API Key: </label>
      <input type="text" v-model="userApi" id="userApi" name="userApi" />
    </div>

    <div>
      <label for="factionFilter">Faction filter: </label>
      <input type="text" v-model="factionSearchData" autocomplete="off" id="factionFilter" name="factionFilter" />
    </div>

    <div class="inoperative">
      <!-- change to range selector -->
      <label for="revDateRangeStart">Revive Date Range Start: </label>
      <input type="date" id="revDateRangeStart" name="revDateRangeStart" />
    </div>
    <div class="inoperative">
      <label for="revDateRangeEnd">Revive Date Range End: </label>
      <input type="date" id="revDateRangeEnd" name="revDateRangeEnd" />
    </div>

    <div>
      <button @click="condensedUserData">Get Data!</button>
    </div>
  </div>

  <div class="loading-bar-wrapper">
    <div v-if="state === 'loading'">
      <div class="lds-ellipsis">
        <div></div>
        <div></div>
        <div></div>
        <div></div>
      </div>
    </div>
  </div>

  <div v-if="state === 'error'">
    <p>There was an error retriving data or with the server.</p>
  </div>

  <div v-if="state === 'success'">
    <div class="user-card-wrapper">
      <template v-for="item in usersRevivedAsAnArrayFilteredByFaction">
        <div class="user-card">
          <div class="card-container">
            <div>{{ item.name }}[{{ item.id }}]</div>
            <div>Revives: {{ item.reviveTotal }}</div>
            <div>Successes: {{ item.reviveSuccess }}</div>
            <div>Failures: {{ item.reviveFailure }}</div>
            <div class="event-log-box">
              <p v-for="log in item.events"><a target="_blank"
                  :href="`http://www.torn.com/profiles.php?XID=${log.id}`">{{
    log.message
                  }}</a></p>
            </div>
          </div>
        </div>
      </template>
    </div>
  </div>

  <p class="tiny-text">
    Ensure you have an API key entered.<br />
    If faction filter is left empty TRL will search all.<br />
    Make sure faction names are spelled correctly.<br />
    Revive date range is deactivated for the time being.
  </p>

</template>

<style scoped lang="scss">
@import url('https://fonts.googleapis.com/css2?family=Ubuntu:wght@300;400;700&display=swap');

.header {
  align-self: flex-start;
}

.event-log-box {
  height: 200px;
  max-width: 100%;
  overflow: auto;
}

.user-card {
  border-radius: 5px;
  box-shadow: 0 4px 8px 0 rgba(0, 0, 0, 0.2);
  transition: 0.3s;
}

.user-card:hover {
  box-shadow: 0 8px 16px 0 lightskyblue;
}

.card-container {
  padding: 15px 38px;
}

.search-grid>div {
  padding: 2px 10px;
}

.tableGrid {
  display: grid;
  grid-template-columns: repeat(6, auto);
  margin-top: 25px;
}


.tableData>div {
  border-bottom: 3px solid darkgray;
  padding: 10px;
}


.tableData>div:nth-child(1n+7) {
  border-right: 1px solid lightgray;
  ;
}

.tableData>div:nth-child(1n+7) {
  border-bottom: 1px solid lightgrey;
}

.tableData>div:nth-last-child(-n+6) {
  border-bottom: 0;
}

.tableData>div:nth-child(6n+0) {
  border-right: 0;
}

a {
  color: cornflowerblue;
}

.tiny-text {
  color: gray;
  font-size: 12px;
}

.inoperative {
  opacity: 0.3;
}

/* loading animation */

.loading-bar-wrapper {
  display: flex;
  align-items: center;
  justify-content: center;
}

.lds-ellipsis {
  display: inline-block;
  justify-self: center;
  position: relative;
  width: 80px;
  height: 80px;
}

.lds-ellipsis div {
  position: absolute;
  top: 33px;
  width: 13px;
  height: 13px;
  border-radius: 50%;
  background: #cef;
  animation-timing-function: cubic-bezier(0, 1, 1, 0);
}

.lds-ellipsis div:nth-child(1) {
  left: 8px;
  animation: lds-ellipsis1 0.6s infinite;
}

.lds-ellipsis div:nth-child(2) {
  left: 8px;
  animation: lds-ellipsis2 0.6s infinite;
}

.lds-ellipsis div:nth-child(3) {
  left: 32px;
  animation: lds-ellipsis2 0.6s infinite;
}

.lds-ellipsis div:nth-child(4) {
  left: 56px;
  animation: lds-ellipsis3 0.6s infinite;
}

@keyframes lds-ellipsis1 {
  0% {
    transform: scale(0);
  }

  100% {
    transform: scale(1);
  }
}

@keyframes lds-ellipsis3 {
  0% {
    transform: scale(1);
  }

  100% {
    transform: scale(0);
  }
}

@keyframes lds-ellipsis2 {
  0% {
    transform: translate(0, 0);
  }

  100% {
    transform: translate(24px, 0);
  }
}

/* Media queries */
$breakpoint-phone: 576px;
$breakpoint-tablet: 768px;
$breakpoint-laptop: 992px;
$breakpoint-desktop: 1200px;

// .screen-size {
//   position: fixed;
//   top: 0;
//   left: 0;

//   width: 100vw;
//   height: 4px;
//   background-color: blue;

//   @media (min-width: $breakpoint-phone) {
//     background-color: red;
//   }

//   @media (min-width: $breakpoint-tablet) {
//     background-color: purple;
//   }

//   @media (min-width: $breakpoint-laptop) {
//     background-color: lime;
//   }

//   @media (min-width: $breakpoint-desktop) {
//     background-color: deeppink;
//   }
// }

.user-card-wrapper {
  display: grid;
  grid-template-columns: repeat(1, minmax(0, 1fr));
  column-gap: 10px;
  row-gap: 15px;

  @media (min-width: $breakpoint-phone) {
    grid-template-columns: repeat(2, minmax(0, 1fr));
  }

  @media (min-width: $breakpoint-tablet) {
    grid-template-columns: repeat(3, minmax(0, 1fr));
  }

  @media (min-width: $breakpoint-laptop) {
    grid-template-columns: repeat(3, minmax(0, 1fr));
  }

  @media (min-width: $breakpoint-desktop) {
    grid-template-columns: repeat(3, minmax(0, 1fr));
  }
}

.search-grid {
  display: grid;
  grid-template-columns: repeat(1, auto);
  margin-bottom: 20px;

  @media (min-width: $breakpoint-phone) {
    grid-template-columns: repeat(2, auto);
  }

  @media (min-width: $breakpoint-tablet) {
    grid-template-columns: repeat(2, auto);
  }

  @media (min-width: $breakpoint-laptop) {
    grid-template-columns: repeat(2, auto);
  }

  @media (min-width: $breakpoint-desktop) {
    grid-template-columns: repeat(2, auto);
  }
}

.event-log-box {
  height: 200px;
  max-width: 100%;
  overflow: auto;

  @media (min-width: $breakpoint-phone) {
    max-width: 300px;
  }

  @media (min-width: $breakpoint-tablet) {
    max-width: 300px;
  }

  @media (min-width: $breakpoint-laptop) {
    max-width: 300px;
  }

  @media (min-width: $breakpoint-desktop) {
    max-width: 300px;
  }
}
</style>
