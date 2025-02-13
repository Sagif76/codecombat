<script>
import { mapGetters, mapMutations, mapActions } from 'vuex'
import _ from 'lodash'
import moment from 'moment'
import VueTimepicker from 'vue2-timepicker'
import UserSearchComponent from './UserSearchComponent'
import TimeZonePicker from './TimeZonePicker'
import MembersAttendeesComponent from './MembersAttendeesComponent'
import momentTz from 'moment-timezone'
import gcApiHandler from '../../../core/social-handlers/GoogleCalendarHandler'
import convertDateMixin from '../mixins/convertDateMixin'

export default {
  name: 'EditInstanceComponent',
  components: {
    'user-search': UserSearchComponent,
    'members-attendees': MembersAttendeesComponent,
    'time-picker': VueTimepicker,
    'timezone-picker': TimeZonePicker
  },
  data () {
    return {
      isSuccess: false,
      inProgress: false,
      errorMessage: '',
      timeZone: 'America/New_York',
      tzOffset: momentTz.tz('America/New_York').format('Z'),
      instance: null,
      memberAttendees: {},
      myTimeZone: momentTz.tz.guess(),
      startDate: null,
      endDate: null,
      startTime: null,
      endTime: null
    }
  },
  mixins: [
    convertDateMixin
  ],
  methods: {
    ...mapActions('events', [
      'saveInstance'
    ]),
    async syncToGoogleCalendar () {
      try {
        const res = await gcApiHandler.syncInstanceToGC(this.instance, this.propsEvent.googleEventId, this.timeZone)
        console.log('Synced to GC', res)
        noty({ text: 'Synced instance to Google Calendar successfully', type: 'success', layout: 'center', timeout: 3000 })
      } catch (err) {
        console.log('Error syncing to GC:', err)
        noty({ text: 'Error syncing to Google Calendar', type: 'error', timeout: 5000 })
      }
    },
    selectOwner (u) {
      Vue.set(this.instance, 'owner', u._id)
      Vue.set(this.instance.ownerDetails, 'name', u.name)
    },
    toggleMember (m) {
      const bool = this.memberAttendees[m].attendance
      this.$set(this.memberAttendees[m], 'attendance', !bool)
    },
    selectTimeZone (tz) {
      this.timeZone = tz
      this.tzOffset = momentTz.tz(this.timeZone).format('Z')
    },
    updateDescription (desc) {
      this.$set(this.memberAttendees[desc.id], 'description', desc.value)
    },
    async onFormSubmit () {
      this.inProgress = true

      let { errMsg, timeUpdated, startDate, endDate } = this.validateDates({
        initialStartDate: this.propsInstance.startDate,
        initialEndDate: this.propsInstance.endDate
      })
      if (errMsg) {
        this.errorMessage = errMsg
        this.inProgress = false
        return
      }
      this.instance.startDate = startDate
      this.instance.endDate = endDate

      if (!this.instance.owner) {
        this.errorMessage = 'Must set an Owner'
        this.inProgress = false
        return
      }
      this.instance.members = Object.values(this.memberAttendees).map(ma => _.pick(ma, ['userId', 'attendance', 'description']))

      try {
        if (timeUpdated && this.propsEvent.syncedToGC) {
          await this.syncToGoogleCalendar()
          timeUpdated = false
        }
        await this.saveInstance(this.instance)
        this.$emit('save', this.instance.event)
        this.inProgress = false
      } catch (err) {
        this.errorMessage = err?.message
        setTimeout(() => {
          this.inProgress = false
        }, 3000)
      }
    },
    instanceUpdate () {
      this.instance = _.cloneDeep(this.propsInstance)
      this.convertTimesForUI({ startDate: this.instance.startDate, endDate: this.instance.endDate, timeZone: this.timeZone })

      if (new Date() > new Date(this.instance.endDate)) {
        this.$set(this.instance, 'done', true)
      }
      this.memberAttendees = {}
      this.propsEvent.members.forEach(m => {
        if (m.startIndex <= this.instance.index && m.startIndex + m.count > this.instance.index) {
          const existMember = _.find(this.instance.members, { userId: m.userId })
          this.$set(this.memberAttendees, m.userId, _.merge({
            userId: m.userId,
            name: m.name,
            attendance: true,
            description: ''
          }, existMember))
        }
      })
    }
  },
  computed: {
    ...mapGetters({
      propsInstance: 'events/eventPanelInstance',
      propsEvent: 'events/eventPanelEvent'
    }),
    _endTimeHourRange () {
      return [[0, 23]]
    }
  },
  watch: {
    propsInstance () {
      this.instanceUpdate()
    }
  },
  mounted () {
    this.instanceUpdate()
  }
}
</script>

<template>
  <div>
    <div class="tab-label">
      {{ $t('events.edit_instance_tab_desc') }}
    </div>
    <form
      class="edit-instance-form"
      @submit.prevent="onFormSubmit"
      v-if="instance"
    >
      <div class="form-group">
        <label for="name"> {{ $t('events.name') }}</label>
        <input
          :value="propsEvent.name"
          name="name"
          class="form-control"
          type="text"
          disabled
        >
      </div>
      <div class="form-group">
        <label for="description"> {{ $t('events.description') }}</label>
        <input
          :value="propsEvent.description"
          name="description"
          class="form-control"
          type="text"
          disabled
        >
      </div>
      <div class="form-group">
        <label for="done"> {{ $t('events.done') }}</label>
        <input
          v-model="instance.done"
          type="checkbox"
          class="form-control"
          name="done"
          :disabled="new Date(instance.endDate) > new Date()"
        >
      </div>
      <div class="form-group">
        <label for="owner"> {{ $t('events.owner') }}</label>
        <user-search
          :value="instance.ownerDetails?.name || instance.ownerDetails?.firstName"
          :permissions="'onlineTeacher'"
          @select="selectOwner"
        />
      </div>

      <div class="form-group">
        <label for="timeZone"> {{ $t('events.time_zone') }}</label>
        <div class="input-lable">
          {{ $t('events.timezone_tips') + myTimeZone }}
        </div>
        <timezone-picker
          :tz="timeZone"
          @select="selectTimeZone"
        />
      </div>

      <div class="form-group">
        <label for="startDate"> {{ $t('events.start_date') }}</label>
        <input
          v-model="startDate"
          type="date"
          class="form-control"
          name="startDate"
          id="startDate"
        >
      </div>
      <div class="form-group">
        <label for="timeRange"> {{ $t('events.time_range') }}</label>
        <div>
          <time-picker format="hh:mm A" :minute-interval="10" v-model="startTime" />
        </div>
      </div>

      <div class="form-group">
        <label for="endDate"> {{ $t('events.end_date') }}</label>
        <input
          v-model="endDate"
          type="date"
          class="form-control"
          name="endDate"
          id="endDate"
        >
      </div>
      <div class="form-group">
        <label for="timeRange"> {{ $t('events.time_range') }}</label>
        <div>
          <time-picker format="hh:mm A" :minute-interval="10" v-model="endTime" />
        </div>
      </div>

      <div class="form-group">
        <label for="video"> {{ $t('events.video_recording') }}</label>
        <input
          v-model="instance.video"
          type="text"
          class="form-control"
          name="video"
        >
      </div>
      <div class="form-group">
        <label for="members">{{ $t('events.members') }}</label>
        <!-- TODO: select participants -->
        <members-attendees
          :instance="propsInstance"
          :members="memberAttendees"
          @toggle-select="toggleMember"
          @update-description="updateDescription"
        />
      </div>

      <div class="form-group pull-right">
        <span
          v-if="isSuccess"
          class="success-msg"
        >
          {{ $t('teacher.success') }}
        </span>
        <span
          v-if="errorMessage"
          class="error-msg"
        >
          {{ errorMessage }}
        </span>
        <button
          class="btn btn-success btn-lg"
          type="submit"
          :disabled="inProgress"
        >
          {{ $t('common.submit') }}
        </button>
      </div>
    </form>
  </div>
</template>

<style lang="scss" scoped>
@import '~vue2-rrule-generator/dist/vue2-rrule-generator.css';
@import '~vue2-timepicker/dist/VueTimepicker.css';

.tab-label {
  font-size: 15px;
  color: rgba(128, 128, 128, 0.7);
}

.error-msg {
  color: red;
}
</style>
