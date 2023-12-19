<template>
  <div>
    <div v-if="showEventModal" class="event-modal-overlay">
      <div class="modal-content modal-event">
        <div class="modal-header">
          <h4 class="modal-title">
            {{ modalTitle }}
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
              <div class="col-10 p-0">
                <v-select :options="customers" v-model="newEvent.client" label="name" class="customers-select"
                  :placeholder="clientPlaceholder">
                </v-select>
              </div>
              <div class="color-display" :style="{ backgroundColor: newEvent.color.hex }"></div>
            </div>
            <input class="form-control" v-model="newEvent.title" placeholder="Titre">
            <textarea class="form-control" v-model="newEvent.desc" placeholder="Description"
              style="resize:both;"></textarea>
            <div class="modal-bottom">
              <div class="date-range">
                <div class="date-start">
                  <datetime v-model="newEvent.start" minute-interval="15" type="datetime-local" noLabel
                    format="YYYY-MM-DD HH:mm:ss" />
                </div>
                <div class="date-end">
                  <datetime v-model="newEvent.end" minute-interval="15" type="datetime-local" noLabel
                    format="YYYY-MM-DD HH:mm:ss" />
                </div>
              </div>
              <div class="modal-checkboxes">
                <div class="modal-checkbox">
                  <input type="checkbox" v-model="newEvent.iswritten">
                  <label for="iswritten">Rédigé</label>
                </div>
                <div class="modal-checkbox">
                  <input type="checkbox" v-model="newEvent.isprog">
                  <label for="isprog">Programmé</label>
                </div>
              </div>

            </div>
            <div class="modal-buttons">
              <button class="card-label-red btn modal-button" @click="deleteEvent">
                <i class="fas fa-trash"></i>
                Supprimer
              </button>
              <button class="card-label-blue btn modal-button" @click="saveEvent">
                <i class="fas fa-save"></i>
                Enregistrer</button>
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
        eventDrop: this.handleEventDrop,
        eventDidMount: this.eventDidMount
      },
      showEventModal: false,
      modalTitle: '',
      newEvent: {
        id: '',
        client: '',
        title: '',
        desc: '',
        start: '',
        end: '',
        color: '',
        iswritten: false,
        isprog: false
      },
      customers: [],
      selectedClient: '',
      editingEvent: null,
    };
  },
  mounted() {
    console.log('Component mounted');
    this.fetchEventsFromBackend();
    this.initializeCalendar();
    this.fetchCustomers();
    this.$refs.fullCalendar.getApi().on('eventClick', this.handleEventClick);
    document.addEventListener('keydown', this.handleEscapeKey);
  },
  beforeDestroy() {
    document.removeEventListener('keydown', this.handleEscapeKey);
  },
  methods: {
    fetchCustomers() {
      axios.get('/customers', { params: { paginate: 1000 } })
        .then((response) => {
          this.customers = response.data.data;
          console.log('(customers:)', this.customers);
        })
        .catch((error) => {
          console.error(error);
        });
    },
    eventDidMount(info) {
      const event = info.event;

      if (event.extendedProps.isprog) {
        info.el.style.outline = '1px ridge #0079bf';
        info.el.style.outlineOffset = '2px';
        info.el.style.margin = '5px 5px';
      }

      if (event.extendedProps.isprog || event.extendedProps.iswritten) {

        // Ajout d'un tooltip au hover
        const tooltip = document.createElement('span');
        tooltip.className = 'tooltipass';
        const tooltipText = [];

        if (event.extendedProps.iswritten) {
          tooltipText.push('✓ Rédigé');
        }
        if (event.extendedProps.isprog) {
          tooltipText.push('✓ Programmé');
        }

        tooltip.innerHTML = tooltipText.join('<br>');


        info.el.appendChild(tooltip);

        // Hide tooltip
        tooltip.style.display = 'none';

        // On hover
        info.el.addEventListener('mouseenter', () => {
          tooltip.style.display = 'block';
        });

        // Out hover
        info.el.addEventListener('mouseleave', () => {
          tooltip.style.display = 'none';
        });

        const tooltipStyle = {
          position: 'absolute',
          zIndex: 9,
          top: '30px',
          left: '45%',
          transform: 'translateX(-50%)',
          background: '#0079bf',
          color: 'white',
          padding: '5px 10px',
          borderRadius: '5px',
          whiteSpace: 'nowrap',
          boxShadow: 'rgb(0 0 0 / 20%) 0px 6px 9px 0px'
        };

        // Loop tooltips style
        Object.keys(tooltipStyle).forEach((prop) => {
          tooltip.style[prop] = tooltipStyle[prop];
        });
      }
    },



    fetchEventsFromBackend() {
      // Fetch events
      return axios
        .get(this.apiUrl)
        .then((response) => {
          console.log('API Response:', response.data);

          // Populate events array
          this.calendarOptions.events = response.data.map((event) => {
            const color = event.color;
            const textColor = this.calculateEventTextColor(color);

            return {
              id: event.id,
              client: event.client,
              title: event.title,
              start: event.start,
              end: event.end,
              description: event.desc,
              color: color,
              textColor: textColor,
              iswritten: event.iswritten,
              isprog: event.isprog
            }
          });

          console.log('Fetched events:', this.calendarOptions.events);

        })
        .catch((error) => {
          console.error(error);
        });
    },
    saveEvent() {
      // Create event object
      const eventData = {
        event_client: this.newEvent.client ? this.newEvent.client.name : '',
        event_name: this.newEvent.title,
        event_desc: this.newEvent.desc,
        event_start: this.$moment(this.newEvent.start).utcOffset(60).format(), // Convertir to UTC+1
        event_end: this.$moment(this.newEvent.end).utcOffset(60).format(),
        event_color: this.newEvent.color.hex, // Store color code
        event_iswritten: this.newEvent.iswritten,
        event_isprog: this.newEvent.isprog
      };

      if (this.editingEvent) {
        // When updating existing
        axios
          .patch(`/calendar/${this.editingEvent.id}`, eventData)
          .then((response) => {
            console.log('Event updated:', response.data);
            console.log('eventPatchedData:', eventData);
            this.editingEvent.setExtendedProp('client', eventData.event_client);
            this.editingEvent.setProp('title', eventData.event_name);
            this.editingEvent.setExtendedProp('description', eventData.event_desc);
            this.editingEvent.setStart(eventData.event_start, { maintainDuration: true });
            const newEnd = new Date(eventData.event_start);
            newEnd.setDate(newEnd.getDate());
            this.editingEvent.setEnd(newEnd.toISOString(), { maintainDuration: true });
            this.editingEvent.setProp('backgroundColor', eventData.event_color);
            this.editingEvent.setExtendedProp('iswritten', eventData.event_iswritten);
            this.editingEvent.setExtendedProp('isprog', eventData.event_isprog);
            this.editingEvent = null;
            this.showEventModal = false;
          })
          .catch((error) => {
            console.error('Error updating event:', error);
          });
      } else {
        // When creating new
        axios
          .post('/calendar', eventData)
          .then((response) => {
            console.log('Event created:', response.data);
            console.log('created new event:', eventData);

            // Set id to avoid error
            const newEvent = {
              id: response.data.id,
              client: eventData.event_client,
              title: eventData.event_name,
              start: eventData.event_start,
              end: eventData.event_end,
              description: eventData.event_desc,
              color: eventData.event_color,
              iswritten: eventData.event_iswritten,
              isprog: eventData.event_isprog
            };

            // Push into events
            this.calendarOptions.events.push(newEvent);

            this.showEventModal = false;

            // Clear fields for next operation
            this.newEvent = {
              client: '',
              title: '',
              desc: '',
              start: '',
              end: '',
              color: '',
              iswritten: false,
              isprog: false
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
      const clientName = event.extendedProps.client;
      if (clientName) {
        const foundClient = this.customers.find(customer => customer.name === clientName);
          if (foundClient) {
            this.newEvent.client = foundClient;
          } else {
            console.warn('Pas de client trouvé pour:', clientName);
            this.newEvent.client = '';
          }
      } else {
        console.warn('Nom du client undefined');
        this.newEvent.client = '';
      }
      this.newEvent.title = event.title;
      this.newEvent.desc = event.extendedProps.description;
      this.newEvent.color = { hex: event.backgroundColor };
      this.newEvent.iswritten = event.extendedProps.iswritten;
      this.newEvent.isprog = event.extendedProps.isprog;
      this.modalTitle = 'Modifier un évènement';
      if (event.start) {
        this.newEvent.start = this.$moment(event.start).format('YYYY-MM-DDTHH:mm');
      }
      if (event.end) {
        this.newEvent.end = this.$moment(event.end).format('YYYY-MM-DDTHH:mm');
      }
      this.showEventModal = true;
    },
    handleDateClick(selectionInfo) {

      if (selectionInfo && selectionInfo.start) {

        this.showEventModal = false;

        const start = new Date(selectionInfo.start);

        const end = new Date(start);

        const formattedStartDate = start.toISOString().slice(0, 16);
        const formattedEndDate = end.toISOString().slice(0, 16);

        this.newEvent.start = formattedStartDate;
        this.newEvent.end = formattedEndDate;

        this.newEvent = {
          id: '',
          client: '',
          title: '',
          desc: '',
          start: formattedStartDate,
          end: formattedEndDate,
        };

        this.modalTitle = 'Ajouter un évènement';

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
          console.log('Event deleted successfully');

          // Remove deleted event from UI
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

      this.modalTitle = 'Ajouter un évènement';

      this.showEventModal = true;
    },
    handleEventDrop(info) {
      const event = info.event;

      if (event.start && event.end) {

        const eventId = event.id;
        const newStart = this.$moment(event.start).utcOffset(60).format();
        const newEnd = this.$moment(event.end).utcOffset(60).format();

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

      if (hexColor && hexColor.length === 7) {
        const r = parseInt(hexColor.slice(1, 3), 16);
        const g = parseInt(hexColor.slice(3, 5), 16);
        const b = parseInt(hexColor.slice(5, 7), 16);

        const colorBrightness = brightness(r, g, b);

        return colorBrightness > 180 ? '#0079bf' : '#fff';
      } else {
      }
    },
    closeEventModal() {
      this.showEventModal = false;
    },
    handleEscapeKey(event) {
      if (event.key === 'Escape') {
        this.closeEventModal();
      }
    },
    initializeCalendar() {
      this.$refs.fullCalendar.$on('loaded', (event) => {
        console.log('FullCalendar loaded event:', event);
      });
      this.$refs.fullCalendar.$on('eventClick', (info) => {
        const event = info.event;

        this.showEventModal = true;
        this.editingEvent = event;

        this.newEvent.client = event.client;
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
      return this.newEvent.color ? this.newEvent.color.hex : '';
    },
    clientPlaceholder() {
      if (this.editingEvent && this.newEvent.client) {
        return this.newEvent.client.name;
      } else {
        return 'Client';
      }
    },
  },
};
</script>


<style>
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
  padding: 0 1.5rem;
  padding-bottom: 1rem;
  align-items: flex-start;
}

.modal-inner {
  display: flex;
  flex-direction: column;
  row-gap: 1rem;
  background: #FFF;
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
  margin: .2rem 0 1rem 0;
}

.modal-button {
  width: 8rem;
}

.modal-bottom {
  display: flex;
  gap: 1rem;
}

.color-display {
  width: 50px;
  height: 15px;
  border: none;
  border-radius: 5px;
}

.modal-checkboxes {
  display: flex;
  flex-direction: column;
  justify-content: flex-end;
}

.modal-checkboxes input[type=checkbox] {
  cursor: pointer;
}

.fc .fc-daygrid-event {
  z-index: unset !important;
}

.date-time-picker {
  width: 12.6rem;
}

.vs--searchable .vs__dropdown-toggle {
  cursor: pointer !important;
}

*[type=search] {
  cursor: pointer;
}
</style>