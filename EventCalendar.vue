<template>
  <div class="event__calendar">
    <div class="cv-header" :locale="params.language">
      <div class="cv-header-nav">
        <button class="previousYear" aria-label="Previous Year" @click.prevent="setShowDate('py')">&laquo;</button>
        <button class="previousPeriod" aria-label="Previous Month" @click.prevent="setShowDate('pm')">&lsaquo;</button>
        <datepicker
          id="filter__date"
          name="filter__date"
          format="dd.MM.yyyy"
          v-model="showDate"
          @selected="selectDate()"
        ></datepicker>
        <button class="nextPeriod" aria-label="Next Month" @click.prevent="setShowDate('nm')">&rsaquo;</button>
        <button class="nextYear" aria-label="Next Year" @click.prevent="setShowDate('ny')">&raquo;</button>
      </div>
      <div v-if="params.allow_mode_switch" class="cv-header-mode">
        <button
          :class="['display-mode', selectedMode == 'list' ? 'active' : '' ]"
          :aria-label="params.translations.list"
          @click.prevent="selectedMode = 'list'; getEvents();"
        ><i class="fa fa-list-ul"> </i> <span>{{ params.translations.list }}</span></button>
        <button
          :class="['display-mode', selectedMode == 'calendar' ? 'active' : '' ]"
          :aria-label="params.translations.calendar"
          @click.prevent="selectedMode = 'calendar'; getEvents();"
        ><i class="fa fa-calendar-alt"></i> <span>{{ params.translations.calendar }}</span></button>
      </div>
    </div>

    <div v-if="selectedMode == 'calendar'">
      <calendar-view
        :items="events"
        :show-date="showDate"
        :starting-day-of-week="1"
        item-top="1.125em"
        item-content-height="1.25em"
        item-border-height="1px"
        class="theme-default"
        :locale="params.language"
      >
        <template v-slot:item="{ value: item, top }">
          <div
            :key="item.id"
            :style="`top:${top}`"
            :class="[item.classes, 'cv-item']"
            :title="item.title"
            @click.stop="onClickEvent(item)"
            v-html="item.originalItem.startTime + item.title"
          />
          <div
            :id="'item_' + item.id"
            :style="`top:${top}`"
            :class="[item.classes, 'cv-item cv-item-details hidden']"
          >
            <b>{{ item.title }}</b>
            <div class="meta">{{ formatDateTime(item.startDate, item.endDate) }}</div>
            <div class="meta" v-if="item.originalItem.location">{{ item.originalItem.location }}</div>
            <div v-if="item.originalItem.description" v-html="item.originalItem.description"></div>
            <div v-if="item.originalItem.url"><a :href="item.originalItem.url">{{ params.translations.moreinfo }}</a></div>
          </div>
        </template>
      </calendar-view>
    </div>

    <div class="event__list" v-if="selectedMode == 'list'" :style="params.cal_height ? `height:${params.cal_height}px` : ''">
      <div class="event__day" v-for="(day, index) in eventsByDate" :key="index">
        <div class="event__date">{{ formatDate(day.date_string) }}</div>
        <div class="event__details" v-for="item in day.events" :key="item.id + '_' + index">
          <div class="event-time">{{ item.startTime }}</div>
          <a class="event-title" @click="toggleContent(item.id + '_' + index)"><h4 class="h6">{{ item.title }}</h4></a>
          <div :id="'event__content_' + item.id + '_' + index" class="event-content hidden">
            <div class="meta">{{ formatDateTime(item.startDate, item.endDate) }}</div>
            <div class="meta" v-if="item.location">{{ item.location }}</div>
            <div class="description" v-if="item.description" v-html="item.description"></div>
            <a class="url" :href="item.url">{{ params.translations.moreinfo }}</a>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { CalendarView } from "vue-simple-calendar"
import Datepicker from 'vuejs-datepicker';
import $ from 'jquery';

import "vue-simple-calendar/static/css/default.css"

export default {
  props: ['params'],

  data() {
    return {
      events: [],
      eventsByDate : [],
      showDate: new Date(),
      selectedMode: this.params.calendar_preset_mode,
    };
  },

  components: {
  	CalendarView,
  	Datepicker,
  },

  mounted: function () {
    this.getEvents();
    document.addEventListener('click', this.handleClickOutside);
  },

  methods: {
    getEvents() {
      // get start date & end date
      var minDate = '';
      var maxDate = '';
      if (this.selectedMode == 'calendar') {
        var currentYear = this.showDate.getFullYear();
        var currentMonth = this.showDate.getMonth();
        minDate = new Date(currentYear, currentMonth - 1, 23).toISOString();
        maxDate = new Date(currentYear, currentMonth + 1, 7).toISOString();
      } else {
        minDate = new Date(this.showDate.getFullYear(), this.showDate.getMonth(), this.showDate.getDate()).toISOString();
        if (this.params.cal_limit_days) {
          maxDate = new Date(this.showDate.getFullYear(), this.showDate.getMonth(), this.showDate.getDate() + 8).toISOString();
        } else {
          maxDate = new Date(this.showDate.getFullYear(), this.showDate.getMonth() + 1, this.showDate.getDate()).toISOString();
        }
      }

      // fetch google calendar response
      var url = 'https://clients6.google.com/calendar/v3/calendars/' + this.params.calendar_email + '/events' +
                '?calendarId=' + encodeURIComponent(this.params.calendar_email) +
                '&singleEvents=true&maxResults=250&sanitizeHtml=true' +
                '&timeMin=' + encodeURIComponent(minDate) + '&timeMax=' + encodeURIComponent(maxDate) +
                '&key=' + this.params.calendar_api_key;

      $.get(url).done((data) => {
        var events = [];
        var events_by_date = [];

        // parse calendar response
        data.items.forEach(item => {
          var has_time = item.end.dateTime ? true : false;
          var start_time = item.start.dateTime ? new Date(item.start.dateTime) : null;
          var start_date = item.start.date ? item.start.date : item.start.dateTime;
          var end_date = item.end.date ? new Date(item.end.date) : null;
          if (end_date) {
            // if event is all day, end date is set to next day in google calendar.
            end_date = end_date.setDate(end_date.getDate() - 1);
          } else {
            end_date = item.end.dateTime;
          }

          var formatted_item = {
            'id': item.id,
            'startDate': start_date,
            'endDate': end_date,
            'title': item.summary,
            'url': item.htmlLink,
            'showTimes': has_time,
            'location': item.location || '',
            'description': item.description || '',
            'startTime': start_time ? ('0' + start_time.getHours()).slice(-2) + ':' + ('0' + start_time.getMinutes()).slice(-2) + ' ' : '',
          }

          if (this.selectedMode == 'list') {
            // add item to list array
            // in list mode, multi-day events are displayed for each date in the event date range
            start_date = new Date(start_date);
            end_date = new Date(end_date);
            var start_date_string = new Date(start_date.getFullYear(), start_date.getMonth(), start_date.getDate()).getTime();
            var end_date_string = new Date(end_date.getFullYear(), end_date.getMonth(), end_date.getDate()).getTime();
            var date_strings = this.getEventDateStrings(start_date_string, end_date_string);
            date_strings.forEach((date_string, i) => {
              var found = false;
              events_by_date.forEach(day => {
                if (date_string == day.date_string) {
                  day.events.push(formatted_item);
                  found = true;
                }
              });
              if (!found) {
                events_by_date.push({ date_string: date_string, events: [ formatted_item ] });
              }
            });
          } else {
            // add item to calendar array
            events.push(formatted_item);
          }
        });

        if (this.selectedMode == 'list') {
          // sort events by time for each day
          events_by_date.forEach(item => {
            if (item.events.length > 1) {
              item.events = this.sortEventArray(item.events);
            }
          });
          // sort dates
          this.eventsByDate = this.sortDateArray(events_by_date);
        } else {
          this.events = events;
        }
      });
    },
    setShowDate(d) {
      // set new date interval and fetch events
      var selectedDate = this.showDate;
      if (this.params.cal_limit_days) {
        var dateInterval = 0;
        switch (d) {
          case 'py':
            dateInterval = -7;
            break;
          case 'pm':
            dateInterval = -1;
            break;
          case 'nm':
            dateInterval = 1;
            break;
          case 'ny':
            dateInterval = 7;
            break;
        }
        selectedDate = new Date(this.showDate.getFullYear(), this.showDate.getMonth(), this.showDate.getDate() + dateInterval);
      } else {
        switch (d) {
          case 'py':
            selectedDate = new Date(this.showDate.getFullYear() - 1, this.showDate.getMonth(), this.showDate.getDate());
            break;
          case 'pm':
            selectedDate = new Date(this.showDate.getFullYear(), this.showDate.getMonth() - 1, this.showDate.getDate());
            break;
          case 'nm':
            selectedDate = new Date(this.showDate.getFullYear(), this.showDate.getMonth() + 1, this.showDate.getDate());
            break;
          case 'ny':
            selectedDate = new Date(this.showDate.getFullYear() + 1, this.showDate.getMonth(), this.showDate.getDate());
            break;
        }
      }
      this.showDate = selectedDate;
      this.getEvents();
    },
    selectDate() {
      this.$nextTick(() => this.setShowDate('custom'));
    },
    onClickEvent(item) {
      // hide all other event detail boxes when a new opens
      $('#item_'+item.id).toggleClass('hidden');
    },
    toggleContent(id) {
      // display / hide the event detail box for a clicked event
      $('#event__content_' + id + ', #event__show_' + id + ', #event__hide_' + id).toggleClass('hidden');
    },
    formatDate(date) {
      date = new Date(date);
      let options = { weekday: 'long', month: 'long', day: 'numeric' };
      return date.toLocaleString(this.params.language, options);
    },
    formatDateTime(startDate, endDate) {
      // format outputted date as e.g. "Tuesday 3 August 18:00-19:00"
      startDate = new Date(startDate);
      endDate = new Date(endDate);
      var formattedDate = '';

      if (startDate.getDate() == endDate.getDate()) {
        let options = { weekday: 'long', month: 'long', day: 'numeric' };
        formattedDate = startDate.toLocaleString(this.params.language, options);
      } else if (startDate.getMonth() == endDate.getMonth()) {
        let options = {month: 'long'};
        formattedDate = startDate.getDate() + '—' + endDate.getDate() + ' ' + startDate.toLocaleString(this.params.language, options);
      } else {
        formattedDate = startDate.getDate() + '.' + (startDate.getMonth()+1) + '—' + endDate.getDate() + (startDate.getMonth()+1);
      }

      var startHours = startDate.getHours();
      if (startHours < 2 || startHours > 3) {
        var endHours = endDate.getHours();
        var startMinutes = startDate.getMinutes();
        var endMinutes = endDate.getMinutes();
        formattedDate = formattedDate + ' ' + ('0' + startHours).slice(-2) + ':' + ('0' + startMinutes).slice(-2);
        if (startHours != endHours || startMinutes != endMinutes) {
          formattedDate = formattedDate + '—' + ('0' + endHours).slice(-2) + ':' + ('0' + endMinutes).slice(-2);
        }
      }

      return formattedDate;
    },
    getEventDateStrings(startDate, endDate) {
      // create an array with all dates in an event date range
      var i, date_strings = [], tempDate;
      for (i = 0; i < 7; i++) {
        date_strings.push(startDate);
        if (startDate == endDate) {
          return date_strings;
        }
        tempDate = new Date(startDate);
        startDate = new Date(tempDate.getFullYear(), tempDate.getMonth(), tempDate.getDate() + 1).getTime();
      }
      return date_strings;
    },
    sortDateArray: function(arr) {
      return arr.slice().sort(function(a, b) {
        if (a.date_string < b.date_string) {
          return -1;
        } else if (a.date_string > b.date_string) {
          return 1;
        }
        return 0;
      });
    },
    sortEventArray: function(arr) {
      return arr.slice().sort(function(a, b) {
        if (a.startTime < b.startTime) {
          return -1;
        } else if (a.startTime > b.startTime) {
          return 1;
        }
        return 0;
      });
    },
    handleClickOutside(evt) {
      // hide all event detail boxes when clicking outside an event details box
      if (!String(evt.target).includes('cv-item-details')) {
        $('.cv-item-details').addClass('hidden');
      }
    },
  },

  beforeDestroy: function () {
    document.removeEventListener('click', this.handleClickOutside);
  }
}
</script>