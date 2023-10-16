<template>
  <div>
    <div v-if="showEventModal" class="event-modal-overlay">
      <div class="modal-content modal-event">
        <div class="modal-header">
          <h4 class="modal-title">
            Ajouter / modifier un évènement
          </h4>
          <button class="close-modal close" @click="closeEventModal">
            <i class="fa fa-times" aria-hidden="true"></i>
          </button>
        </div>
        <div class="modal-colorandevent">
          <div class="color-picker">
            <sketch v-model="newEvent.color" @input="updateSelectedColor"></sketch>
          </div>
          <div class="modal-inner">
            <div class="modal-inner-title">
              <div class="color-display" :style="{ backgroundColor: newEvent.color.hex }"></div>
              <h5>Titre de l'évènement</h5>
            </div>
            <input v-model="newEvent.title">
            <h5>Description</h5>
            <textarea v-model="newEvent.desc"></textarea>
            <div class="modal-bottom">
              <div class="date-range">
                <div class="date-start">
                  <input type="datetime-local" v-model="newEvent.start" placeholder="Event Start">
                </div>
                <div class="date-end">
                  <input type="datetime-local" v-model="newEvent.end" placeholder="Event End">
                </div>
              </div>
              <div class="modal-buttons">
                <button class="card-label-red btn modal-button" @click="deleteEvent">
                  <i class="fas fa-trash"></i>
                  Supprimer
                </button>
                <button class="card-label-blue btn modal-button" @click="saveEvent">
                  <i class="fas fa-save"></i>
                  Enregistrer
                </button>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>
    <div id="calendar-container" style="width: 100%; height: auto;">
      <full-calendar ref="fullCalendar" :options="calendarOptions"></full-calendar>
    </div>
  </div>
</template>

<script>
import FullCalendar from '@fullcalendar/vue'
import dayGridPlugin from '@fullcalendar/daygrid'
import timeGridPlugin from '@fullcalendar/timegrid'
import interactionPlugin from '@fullcalendar/interaction'
import frLocale from '@fullcalendar/core/locales/fr'
import { Sketch } from 'vue-color';

export default {
  components: {
    FullCalendar,
    Sketch,
  },
  props: {
    apiUrl: String,
    events: Array,
  },
  data() {
    return {
      calendarOptions: {
        plugins: [dayGridPlugin, interactionPlugin, timeGridPlugin],
        initialView: 'dayGridMonth',
        headerToolbar: {
          left: 'prev,next today',
          center: 'title',
          right: 'dayGridMonth,timeGridWeek',
        },
        weekends: true,
        events: [],
        eventDisplay: 'block',
        eventColor: this.dynamicEventColor,
        eventTextColor: 'white',
        locale: frLocale,
        dateClick: this.handleDateClick,
        selectable: true,
        select: this.handleDateRangeSelect,
        editable: true,
        eventDrop: this.handleEventDrop
      },
      showEventModal: false,
      newEvent: {
        id: '',
        title: '',
        desc: '',
        start: '',
        end: '',
        color: ''
      },
      editingEvent: null,
    };
  },
  mounted() {
    console.log('Component mounted');
    this.fetchEventsFromBackend();
    this.initializeCalendar();
    this.$refs.fullCalendar.getApi().on('eventClick', this.handleEventClick);
  },
  methods: {
    fetchEventsFromBackend() {
      return axios
        .get(this.apiUrl)
        .then((response) => {
          this.calendarOptions.events = response.data.map((event) => {
            const color = event.color;
            const textColor = this.calculateEventTextColor(color);
            return {
              id: event.id,
              title: event.title,
              start: event.start,
              end: event.end,
              description: event.desc,
              color: color,
              textColor: textColor,
            }
          });
        })
        .catch((error) => {
          console.error(error);
        });
    },
    saveEvent() {
      const eventData = {
        event_name: this.newEvent.title,
        event_desc: this.newEvent.desc,
        event_start: this.$moment(this.newEvent.start).utcOffset(120).format(),
        event_end: this.$moment(this.newEvent.end).utcOffset(120).format(),
        event_color: this.newEvent.color.hex
      };

      if (this.editingEvent) {
        axios
          .patch(`/calendar/${this.editingEvent.id}`, eventData)
          .then((response) => {
            this.editingEvent.setProp('title', eventData.event_name);
            this.editingEvent.setExtendedProp('description', eventData.event_desc);
            this.editingEvent.setStart(eventData.event_start, { maintainDuration: true });
            const newEnd = new Date(eventData.event_start);
            newEnd.setDate(newEnd.getDate() + 1);
            this.editingEvent.setEnd(newEnd.toISOString(), { maintainDuration: true });
            this.editingEvent.setProp('backgroundColor', eventData.event_color);
            this.editingEvent = null;
            this.showEventModal = false;
          })
          .catch((error) => {
            console.error('Error updating event:', error);
          });
      } else {
        axios
          .post('/calendar', eventData)
          .then((response) => {
            const newEvent = {
              id: response.data.id,
              title: eventData.event_name,
              start: eventData.event_start,
              end: eventData.event_end,
              description: eventData.event_desc,
              color: eventData.event_color
            };
            this.calendarOptions.events.push(newEvent);
            this.showEventModal = false;
            this.newEvent = {
              title: '',
              desc: '',
              start: '',
              end: '',
              color: ''
            };
          })
          .catch((error) => {
            console.error('Error creating event:', error);
          });
      }
    },
    handleEventClick(info) {
      const event = info.event;
      this.editingEvent = event;
      this.newEvent.title = event.title;
      this.newEvent.desc = event.extendedProps.description;
      this.newEvent.color = { hex: event.backgroundColor };
      if (event.start) {
        this.newEvent.start = this.$moment(event.start).utcOffset(120).format('YYYY-MM-DDTHH:mm');
      }
      if (event.end) {
        this.newEvent.end = this.$moment(event.end).utcOffset(120).format('YYYY-MM-DDTHH:mm');
      }
      this.showEventModal = true;
    },
    handleDateClick(selectionInfo) {
      if (selectionInfo && selectionInfo.start) {
        this.showEventModal = false;
        const start = new Date(selectionInfo.start);
        start.setHours(8, 0, 0);
        const end = new Date(start);
        end.setHours(18, 0, 0);
        const formattedStartDate = start.toISOString().slice(0, 16);
        const formattedEndDate = end.toISOString().slice(0, 16);
        this.newEvent.start = formattedStartDate;
        this.newEvent.end = formattedEndDate;
        this.newEvent = {
          id: '',
          title: '',
          desc: '',
          start: formattedStartDate,
          end: formattedEndDate,
        };
        this.showEventModal = true;
      }
      this.editingEvent = null;
    },
    deleteEvent() {
      if (!this.editingEvent) {
        return;
      }
      const eventId = this.editingEvent.id;
      axios
        .delete(`/calendar/${eventId}`)
        .then(() => {
          this.removeEventFromCalendar(eventId);
          this.showEventModal = false;
        })
        .catch((error) => {
          console.error('Error deleting event:', error);
        });
    },
    removeEventFromCalendar(eventId) {
      const calendarApi = this.$refs.fullCalendar.getApi();
      calendarApi.getEventById(eventId)?.remove();
    },
    handleDateRangeSelect(selectionInfo) {
      const start = selectionInfo.startStr;
      const end = selectionInfo.endStr;
      const formattedStartDate = new Date(start).toISOString().slice(0, 16);
      const formattedEndDate = new Date(end).toISOString().slice(0, 16);
      this.newEvent.start = formattedStartDate;
      this.newEvent.end = formattedEndDate;
      this.showEventModal = true;
    },
    handleEventDrop(info) {
      const event = info.event;
      if (event.start && event.end) {
        const eventId = event.id;
        const newStart = this.$moment(event.start).utcOffset(120).format();
        const newEnd = this.$moment(event.end).utcOffset(120).format();
        const foundEvent = this.calendarOptions.events.find((e) => e.id === eventId);
        if (foundEvent) {
          foundEvent.start = newStart;
          foundEvent.end = newEnd;
        }
        axios
          .patch(`/calendar/${eventId}/update-times`, {
            event_start: newStart,
            event_end: newEnd,
          })
          .then((response) => {
            console.log('Event updated:', response.data);
          })
          .catch((error) => {
            console.error('Error updating event:', error);
          });
      }
    },
    calculateEventTextColor(hexColor) {
      function brightness(r, g, b) {
        return (r * 299 + g * 587 + b * 114) / 1000;
      }
      const r = parseInt(hexColor.slice(1, 3), 16);
      const g = parseInt(hexColor.slice(3, 5), 16);
      const b = parseInt(hexColor.slice(5, 7), 16);
      const colorBrightness = brightness(r, g, b);
      return colorBrightness > 180 ? '#0079bf' : '#fff';
    },
    closeEventModal() {
      this.showEventModal = false;
    },
    initializeCalendar() {
      this.$refs.fullCalendar.$on('loaded', (event) => {
        console.log('FullCalendar loaded event:', event);
        const calendarInstance = this.$refs.fullCalendar.instanceRef;
        if (calendarInstance) {
          console.log('Found calendar instance.');
          console.log('Events before any changes:', calendarInstance.getApi().getEvents());
          calendarInstance.getApi().destroy();
          calendarInstance.getApi().setOption('events', this.calendarOptions.events);
          calendarInstance.getApi().render();
          console.log('Events after changes:', calendarInstance.getApi().getEvents());
        } else {
          console.log('Failure: Calendar instance not found');
        }
      });
      this.$refs.fullCalendar.$on('eventClick', (info) => {
        const event = info.event;
        this.showEventModal = true;
        this.editingEvent = event;
        this.newEvent.title = event.title;
        this.newEvent.desc = event.desc;
        this.newEvent.start = event.startStr;
        this.newEvent.end = event.endStr;
      });
    },
    updateSelectedColor(color) {
      this.newEvent.color = color;
    },
  },
  computed: {
    dynamicEventColor() {
      return this.newEvent.color ? this.newEvent.color.hex : 'green';
    },
  },
};
</script>

<style scoped>
.event-modal-overlay {
  position: fixed;
  width: 100%;
  height: 100%;
  background: #80808021;
  backdrop-filter: blur(2px);
  z-index: 99;
  display: flex;
  align-items: center;
  justify-content: center;
}

.modal-event {
  position: relative;
  display: flex;
  flex-direction: column;
  background: #FFF;
  z-index: 99;
  gap: 1rem;
  width: auto;
  transform: translateX(-190px);
}

.modal-header {
  padding: 1.5rem !important;
}

.modal-colorandevent {
  display: flex;
  margin: auto;
  gap: 1rem;
  padding: 0 1rem;
  align-items: center;
}

.modal-inner {
  display: flex;
  flex-direction: column;
  row-gap: 1rem;
  background: #FFF;
  padding: 2rem;
}

.modal-inner-title {
  display: flex;
  align-items: center;
  gap: .5rem;
}

.modal-inner-title h5 {
  margin: 0;
}

.date-range {
  display: flex;
  flex-direction: column;
  flex-basis: 45%;
  gap: .4rem;
  justify-content: flex-end;
}

.date-start {
  display: flex;
  align-items: center;
  gap: .4rem;
}

.date-start label {
  margin: 0;
  width: 4rem;
}

.date-end {
  display: flex;
  align-items: center;
  gap: .4rem;
}

.date-end label {
  margin: 0;
  width: 4rem;
}

.modal-buttons {
  display: flex;
  flex-basis: 45%;
  align-items: center;
  gap: 1rem;
  margin: 2rem -1rem 1rem 1rem;
}

.modal-button {
  width: 8rem;
}

.modal-bottom {
  display: flex;
  justify-content: space-between;
  height: 5rem;
}

.color-display {
  width: 15px;
  height: 15px;
  border: none;
  border-radius: 100%;
}
</style>
