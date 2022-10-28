<script>
import { objectToString } from '@vue/shared';
import ky from 'ky';
export default {
  data() {
    return {
      userReviveData: {},
      userEventsData: {},
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
          events: this.parsedUserEventsAsAnArray.filter(log => log.id === item.id).map(log => log.message)
        }
      })
    },

    usersEventsAsAnArray() {
      return Object.values(this.userEventsData).map(item => item.event)
    },

    parsedUserEventsAsAnArray() {
        const regex = /XID\=([0-9]+)(?:"\>|\>)(.+)\<.*/gm;
        return this.usersEventsAsAnArray
            .map(item => ({
                match: regex.exec(item),
                message: item
            }))
            .filter(item => item.match !== null)
            .map(item => {
                return {
                    id: parseInt(item.match[1]),
                    message: item.message
                }
            })
      }

  },

  methods: {


    async getUserReviveData() {
      const { revives } = await ky.get(`https://api.torn.com/user/${import.meta.env.VITE_TORN_USER_ID}`, {
        searchParams:
        {
          key: import.meta.env.VITE_TORN_API_TOKEN,
          selections: 'revives',
        }
      }).json();
      return revives;
    },

    async getUserEventData() {
      const { events } = await ky.get(`https://api.torn.com/user/${import.meta.env.VITE_TORN_USER_ID}`, {
        searchParams:
        {
          key: import.meta.env.VITE_TORN_API_TOKEN,
          selections: 'events',
        }
      }).json();
      return events;
    },

    async condensedUserData() {
      try {
        const [revives, events] = await Promise.all([this.getUserReviveData(), this.getUserEventData()])
        this.userReviveData = revives;
        this.userEventsData = events;
      } catch (err) {
        console.error("Get all data failed", err)
      }
    }
  }
}

</script>

<template>
  <!-- <pre>{{ parsedUserEventsAsAnArray }}</pre> -->
  <header class="header">
    <h1>Torn Revive Ledger</h1>
  </header>
  <div class="search-grid">
    <!-- <div>
      <label for="yourID">Your ID: </label>
      <input type="text" id="yourID" name="yourID" />
    </div> -->
    <div>
      <label for="yourAPI">Your API Key: </label>
      <input type="text" id="yourAPI" name="yourAPI" />
    </div>

    <div>
      <label for="factionFilter">Faction filter: </label>
      <input type="text" id="factionFilter" name="factionFilter" />
    </div>

    <div>
      <!-- change to range selector -->
      <label for="revDateRangeStart">Revive Date Range Start: </label>
      <input type="date" id="revDateRangeStart" name="revDateRangeStart" />
    </div>
    <div>
      <label for="revDateRangeEnd">Revive Date Range End: </label>
      <input type="date" id="revDateRangeEnd" name="revDateRangeEnd" />
    </div>

    <div>
      <button @click="condensedUserData">Get Data!</button>
    </div>
  </div>

  <!-- <div class="tableGrid tableData">
      <div>Revive Target</div>
      <div>Target Faction</div>
      <div>Revive Chance</div>
      <div>Hospital Reason</div>
      <div>Result</div>
      <div>Online</div>
      <template v-for="item in userReviveDataAsAnArray">
        <div>{{ item.target_name }}</div>
        <div>{{ item.target_factionname }}</div>
        <div>{{ item.chance }}</div>
        <div>{{ item.target_hospital_reason }}</div>
        <div>{{ item.result }}</div>
        <div>{{ item.target_last_action.status }}</div>
      </template>
    </div> -->

  <div class="user-card-wrapper">
    <template v-for="item in usersRevivedAsAnArray">
      <div class="user-card">
        <div class="card-container">
          <div>{{ item.name }}[{{ item.id }}]</div>
          <div>Revives: {{ item.reviveTotal }}</div>
          <div>Successes: {{ item.reviveSuccess }}</div>
          <div>Failures: {{ item.reviveFailure }}</div>
          <div class="event-log-box">
            <p v-for="log in item.events">{{ log }}</p>
          </div>
        </div>
      </div>
    </template>
  </div>

  <!-- <div class="user-card-wrapper">
      <div class="user-card">
        <h3>Tempo[123456]</h3>
        <p>
          revives: 14
          successes: 10
          failures: 4
        </p>
      </div>
    </div> -->

  <!-- <div class="faction-card-wrapper">
      <div class="faction-card">
        Faction Card
      </div>
    </div> -->
</template>

<style scoped>
.header {
  align-self: flex-start;
}

.event-log-box {
  height: 200px;
  max-width: 200px;
  overflow: auto;
}

.user-card-wrapper {
  display: grid;
  grid-template-columns: repeat(5, auto);
  column-gap: 10px;
  row-gap: 15px;
}

.user-card {
  border-radius: 5px;
  box-shadow: 0 4px 8px 0 rgba(0, 0, 0, 0.2);
  transition: 0.3s;
}

.user-card:hover {
  box-shadow: 0 8px 16px 0 rgba(0, 0, 0, 0.2);
}

.card-container {
  padding: 15px 38px;
}

.search-grid {
  display: grid;
  grid-template-columns: repeat(2, auto);
  margin-bottom: 20px;
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
</style>
