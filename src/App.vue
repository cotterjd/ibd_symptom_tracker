<template>
  <div class="app">
    <h1 class="title">Symptom Tracker</h1>

    <!-- Calendar header: month navigation -->
    <div class="calendar-header">
      <button class="nav-btn" @click="prevMonth">&#8249;</button>
      <span class="month-label">{{ monthLabel }}</span>
      <button class="nav-btn" @click="nextMonth">&#8250;</button>
    </div>

    <!-- Day-of-week labels -->
    <div class="day-labels">
      <span v-for="d in dayNames" :key="d" class="day-name">{{ d }}</span>
    </div>

    <!-- Calendar grid -->
    <div class="calendar-grid">
      <!-- Leading empty cells for first day of month offset -->
      <div
        v-for="n in firstDayOffset"
        :key="'empty-' + n"
        class="calendar-cell empty"
      ></div>

      <!-- Day cells -->
      <div
        v-for="day in daysInMonth"
        :key="day"
        class="calendar-cell"
        :class="dayColor(day)"
        :style="{ opacity: isFutureDay(day) ? 0.4 : 1, cursor: isFutureDay(day) ? 'default' : 'pointer' }"
        @click="openModal(day)"
      >
        <span class="day-number">{{ day }}</span>
        <span v-if="getBmCount(day) !== null" class="bm-count">{{ getBmCount(day) }}</span>
      </div>
    </div>

    <!-- Symptoms list -->
    <div class="symptoms-list">
      <!-- Legend -->
      <div class="legend">
        <div class="legend-item"><span class="swatch green"></span> No symptoms</div>
        <div class="legend-item"><span class="swatch yellow"></span> 1 symptom</div>
        <div class="legend-item"><span class="swatch orange"></span> 2 symptoms</div>
        <div class="legend-item"><span class="swatch red"></span> 3 symptoms</div>
      </div>

      <p class="symptoms-list-eyebrow">Symptoms</p>
      <ul class="symptoms-items">
        <li class="symptom-item">Blood in stool</li>
        <li class="symptom-item">Mucus in stool</li>
        <li class="symptom-item">5 or more stools</li>
      </ul>
    </div>

    <!-- Version footer -->
    <div class="version-footer">v{{ version }}</div>

    <!-- Modal overlay -->
    <div v-if="modalOpen" class="modal-overlay" @click.self="closeModal">
      <div class="modal">
        <h2 class="modal-title">{{ modalTitle }}</h2>

        <div class="form-field">
          <label>How many BMs did you have today?</label>
          <input
            v-model.number="form.bmCount"
            type="number"
            min="0"
            class="number-input"
          />
        </div>

        <div class="form-field">
          <label>Did any of them include mucus?</label>
          <div class="radio-group">
            <label class="radio-label">
              <input type="radio" v-model="form.mucus" :value="true" /> Yes
            </label>
            <label class="radio-label">
              <input type="radio" v-model="form.mucus" :value="false" /> No
            </label>
          </div>
        </div>

        <div class="form-field">
          <label>Did any of them include blood (not just on the toilet paper)?</label>
          <div class="radio-group">
            <label class="radio-label">
              <input type="radio" v-model="form.blood" :value="true" /> Yes
            </label>
            <label class="radio-label">
              <input type="radio" v-model="form.blood" :value="false" /> No
            </label>
          </div>
        </div>

        <div class="modal-actions">
          <button class="btn-save" @click="saveDay">Save</button>
          <button class="btn-cancel" @click="closeModal">Cancel</button>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { version } from '../package.json'

const STORAGE_KEY = `symptom-tracker-data`

function storageGet() {
  try {
    return JSON.parse(localStorage.getItem(STORAGE_KEY) || `{}`)
  } catch {
    return {}
  }
}

function storageSet(data) {
  localStorage.setItem(STORAGE_KEY, JSON.stringify(data))
}

function dateKey(year, month, day) {
  return `${year}-${String(month + 1).padStart(2, `0`)}-${String(day).padStart(2, `0`)}`
}

export default {
  data() {
    const now = new Date()
    return {
      currentYear: now.getFullYear(),
      currentMonth: now.getMonth(), // 0-indexed
      dayNames: [`Sun`, `Mon`, `Tue`, `Wed`, `Thu`, `Fri`, `Sat`],
      modalOpen: false,
      selectedDay: null,
      form: {
        bmCount: 0,
        mucus: false,
        blood: false,
      },
      data: storageGet(),
      version,
    }
  },
  computed: {
    monthLabel() {
      const d = new Date(this.currentYear, this.currentMonth, 1)
      return d.toLocaleString(`default`, { month: `long`, year: `numeric` })
    },
    daysInMonth() {
      return new Date(this.currentYear, this.currentMonth + 1, 0).getDate()
    },
    firstDayOffset() {
      return new Date(this.currentYear, this.currentMonth, 1).getDay()
    },
    modalTitle() {
      if (this.selectedDay === null) return ``
      const d = new Date(this.currentYear, this.currentMonth, this.selectedDay)
      return d.toLocaleDateString(`default`, { weekday: `long`, month: `long`, day: `numeric`, year: `numeric` })
    },
  },
  methods: {
    prevMonth() {
      if (this.currentMonth === 0) {
        this.currentMonth = 11
        this.currentYear -= 1
      } else {
        this.currentMonth -= 1
      }
    },
    nextMonth() {
      if (this.currentMonth === 11) {
        this.currentMonth = 0
        this.currentYear += 1
      } else {
        this.currentMonth += 1
      }
    },
    isFutureDay(day) {
      const now = new Date()
      const today = new Date(now.getFullYear(), now.getMonth(), now.getDate())
      const cell = new Date(this.currentYear, this.currentMonth, day)
      return cell > today
    },
    getDayData(day) {
      const key = dateKey(this.currentYear, this.currentMonth, day)
      return this.data[key] || null
    },
    getBmCount(day) {
      const d = this.getDayData(day)
      return d !== null ? d.bmCount : null
    },
    dayColor(day) {
      if (this.isFutureDay(day)) return ``
      const d = this.getDayData(day)
      if (!d) return ``
      let score = 0
      if (d.bmCount >= 5) score++
      if (d.mucus) score++
      if (d.blood) score++
      if (score === 0) return `green`
      if (score === 1) return `yellow`
      if (score === 2) return `orange`
      return `red`
    },
    openModal(day) {
      if (this.isFutureDay(day)) return
      this.selectedDay = day
      const existing = this.getDayData(day)
      if (existing) {
        this.form = { ...existing }
      } else {
        this.form = { bmCount: 0, mucus: false, blood: false }
      }
      this.modalOpen = true
    },
    closeModal() {
      this.modalOpen = false
      this.selectedDay = null
    },
    saveDay() {
      if (this.selectedDay === null) return
      const key = dateKey(this.currentYear, this.currentMonth, this.selectedDay)
      this.data = {
        ...this.data,
        [key]: { ...this.form },
      }
      storageSet(this.data)
      this.closeModal()
    },
  },
}
</script>

<style scoped>
.app {
  padding-bottom: 40px;
}

.title {
  text-align: center;
  font-size: 1.4rem;
  margin-bottom: 16px;
  color: #ffffff;
}

/* Calendar header */
.calendar-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 12px;
}

.nav-btn {
  background: none;
  border: 1px solid #555;
  color: #e0e0e0;
  font-size: 1.6rem;
  width: 40px;
  height: 40px;
  border-radius: 6px;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  line-height: 1;
}

.nav-btn:hover {
  background: #333;
}

.month-label {
  font-size: 1.1rem;
  font-weight: 600;
}

/* Day-of-week header */
.day-labels {
  display: grid;
  grid-template-columns: repeat(7, 1fr);
  margin-bottom: 4px;
}

.day-name {
  text-align: center;
  font-size: 0.75rem;
  color: #aaa;
  padding: 4px 0;
}

/* Calendar grid */
.calendar-grid {
  display: grid;
  grid-template-columns: repeat(7, 1fr);
  gap: 4px;
}

.calendar-cell {
  aspect-ratio: 1;
  border-radius: 6px;
  background: #2a2a3e;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  user-select: none;
  transition: filter 0.15s;
  position: relative;
}

.calendar-cell:not(.empty):hover {
  filter: brightness(1.2);
}

.calendar-cell.empty {
  background: transparent;
  cursor: default;
}

.day-number {
  font-size: 0.85rem;
  font-weight: 600;
}

.bm-count {
  font-size: 0.7rem;
  color: rgba(255, 255, 255, 0.85);
  margin-top: 2px;
}

/* Color classes */
.calendar-cell.green {
  background: #2d7a2d;
}
.calendar-cell.yellow {
  background: #8a7a00;
}
.calendar-cell.orange {
  background: #a05000;
}
.calendar-cell.red {
  background: #8a1a1a;
}

/* Legend */
.legend {
  display: flex;
  flex-wrap: wrap;
  gap: 8px 16px;
  margin-top: 0px;
  margin-bottom: 16px;
  justify-content: center;
}

.legend-item {
  display: flex;
  align-items: center;
  gap: 6px;
  font-size: 0.8rem;
  color: #ccc;
}

.swatch {
  display: inline-block;
  width: 14px;
  height: 14px;
  border-radius: 3px;
}

.swatch.green { background: #2d7a2d; }
.swatch.yellow { background: #8a7a00; }
.swatch.orange { background: #a05000; }
.swatch.red { background: #8a1a1a; }

/* Symptoms list */
.symptoms-list {
  margin-top: 22px;
  padding: 16px;
  border: 1px solid rgba(255, 255, 255, 0.06);
  border-radius: 12px;
  background: rgba(255, 255, 255, 0.03);
  color: #ccc;
  text-align: center;
}

.symptoms-list-eyebrow {
  margin-bottom: 10px;
  font-size: 0.7rem;
  font-weight: 600;
  letter-spacing: 0.12em;
  text-transform: uppercase;
  color: #9ea6ba;
}

.symptoms-items {
  list-style: none;
  display: grid;
  gap: 8px;
  justify-items: center;
  padding: 0;
}

.symptom-item {
  width: min(100%, 260px);
  padding: 8px 14px;
  border-radius: 999px;
  background: rgba(255, 255, 255, 0.04);
  border: none;
  color: #d8dcea;
  font-size: 0.84rem;
  font-weight: 500;
  line-height: 1.4;
}

/* Version footer */
.version-footer {
  margin-top: 24px;
  text-align: center;
  font-size: 0.75rem;
  color: #666;
}

/* Modal */
.modal-overlay {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.7);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 200;
  padding: 16px;
}

.modal {
  background: #1e1e30;
  border-radius: 12px;
  padding: 24px;
  width: 100%;
  max-width: 420px;
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.5);
}

.modal-title {
  font-size: 1rem;
  font-weight: 600;
  margin-bottom: 20px;
  color: #fff;
  line-height: 1.4;
}

.form-field {
  margin-bottom: 20px;
}

.form-field label {
  display: block;
  font-size: 0.9rem;
  margin-bottom: 8px;
  color: #ddd;
  line-height: 1.4;
}

.number-input {
  width: 100%;
  padding: 10px 14px;
  background: #2a2a3e;
  border: 1px solid #555;
  border-radius: 6px;
  color: #e0e0e0;
  font-size: 1rem;
}

.number-input:focus {
  outline: none;
  border-color: #7a7af0;
}

.radio-group {
  display: flex;
  gap: 20px;
}

.radio-label {
  display: flex;
  align-items: center;
  gap: 6px;
  font-size: 0.9rem;
  cursor: pointer;
  color: #ddd;
}

.radio-label input[type="radio"] {
  width: 18px;
  height: 18px;
  cursor: pointer;
  accent-color: #7a7af0;
}

.modal-actions {
  display: flex;
  gap: 12px;
  margin-top: 24px;
}

.btn-save, .btn-cancel {
  flex: 1;
  padding: 12px;
  border: none;
  border-radius: 8px;
  font-size: 1rem;
  cursor: pointer;
  font-weight: 600;
}

.btn-save {
  background: #7a7af0;
  color: #fff;
}

.btn-save:hover {
  background: #6060e0;
}

.btn-cancel {
  background: #3a3a4e;
  color: #ddd;
}

.btn-cancel:hover {
  background: #4a4a5e;
}
</style>
