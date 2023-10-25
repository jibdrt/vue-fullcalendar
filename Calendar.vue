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
          console.log(this.customers);
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

        // Hide le tooltip
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

        // Loop pour style les tooltips
        Object.keys(tooltipStyle).forEach((prop) => {
          tooltip.style[prop] = tooltipStyle[prop];
        });
      }
    },



    fetchEventsFromBackend() {
      // Fetch les events du back-end
      return axios
        .get(this.apiUrl)
        .then((response) => {
          console.log('API Response:', response.data);

          // Populate array event avec les données de chaque item
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
      // Créer un object pour représenter l'évènement
      const eventData = {
        event_client: this.newEvent.client.name,
        event_name: this.newEvent.title,
        event_desc: this.newEvent.desc,
        event_start: this.$moment(this.newEvent.start).utcOffset(120).format(), // Convertir to UTC+2
        event_end: this.$moment(this.newEvent.end).utcOffset(120).format(),
        event_color: this.newEvent.color.hex, // Stocker le code couleur
        event_iswritten: this.newEvent.iswritten,
        event_isprog: this.newEvent.isprog
      };

      if (this.editingEvent) {
        // Dans le cas où on update un event existant
        axios
          .patch(`/calendar/${this.editingEvent.id}`, eventData)
          .then((response) => {
            console.log('Event updated:', response.data);
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
        // Dans le cas où on crée un nouvel event
        axios
          .post('/calendar', eventData)
          .then((response) => {
            console.log('Event created:', response.data);

            // Set l'ID pour éviter erreurs
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

            // Push sur le calendrier
            this.calendarOptions.events.push(newEvent);

            // Cacher la modal
            this.showEventModal = false;

            // Clear les fields pour manip suivante
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
      this.newEvent.client = event.extendedProps.client;
      this.newEvent.title = event.title;
      this.newEvent.desc = event.extendedProps.description;
      this.newEvent.color = { hex: event.backgroundColor };
      this.newEvent.iswritten = event.extendedProps.iswritten;
      this.newEvent.isprog = event.extendedProps.isprog;
      this.modalTitle = 'Modifier un évènement';

      console.log(event.client);

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
        // Clear les champs TODO

        this.showEventModal = false;

        // Set le départ en fonction du jour cliqué
        const start = new Date(selectionInfo.start);

        // Set the start time to 8:00 AM
        start.setHours(8, 0, 0); // TODO - ne fonctionne pas

        const end = new Date(start);

        // TODO - ne fonctionne pas
        end.setHours(18, 0, 0);

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

      const eventId = this.editingEvent.id; // Récupérer l'ID de l'event concerné

      // Delete en back et DB
      axios
        .delete(`/calendar/${eventId}`)
        .then(() => {
          console.log('Event deleted successfully');

          // Remove the deleted event from the UI
          this.removeEventFromCalendar(eventId);

          // Close the modal
          this.showEventModal = false;
        })
        .catch((error) => {
          console.error('Error deleting event:', error);
          // Handle the error
        });
    },
    removeEventFromCalendar(eventId) {
      const calendarApi = this.$refs.fullCalendar.getApi();
      calendarApi.getEventById(eventId)?.remove();
    },
    handleDateRangeSelect(selectionInfo) {
      // Récupérer le début et fin depuis la séléction
      const start = selectionInfo.startStr;
      const end = selectionInfo.endStr;

      // Formattage pour les input datetime-local de la modal
      const formattedStartDate = new Date(start).toISOString().slice(0, 16);
      const formattedEndDate = new Date(end).toISOString().slice(0, 16);

      // Set le début et fin pour la création du nouvel event
      this.newEvent.start = formattedStartDate;
      this.newEvent.end = formattedEndDate;

      this.modalTitle = 'Ajouter un évènement';

      // Ouvrir la modal
      this.showEventModal = true;
    },
    handleEventDrop(info) {
      const event = info.event;

      if (event.start && event.end) {

        const eventId = event.id;
        const newStart = this.$moment(event.start).utcOffset(120).format(); // Convertir vers UTC+2 (éviter le moins 2 heures à chaque update d'évènement)
        const newEnd = this.$moment(event.end).utcOffset(120).format();

        // Trouver l'event dans l'array et update
        const foundEvent = this.calendarOptions.events.find((e) => e.id === eventId);
        if (foundEvent) {
          foundEvent.start = newStart;
          foundEvent.end = newEnd;
        }

        // Requête pour mettre à jour en base
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
        // Close the modal
        this.closeEventModal();
      }
    },
    initializeCalendar() {
      this.$refs.fullCalendar.$on('loaded', (event) => {
        console.log('FullCalendar loaded event:', event);
      });
      this.$refs.fullCalendar.$on('eventClick', (info) => {
        const event = info.event;

        // Ouvrir la modal pour éditer
        this.showEventModal = true;

        // Stocker l'event cliqué pour l'éditer
        this.editingEvent = event;

        // Remplir les champs avec les données de l'édition
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