<template>
  <table
    role="grid"
    :aria-label="t('el.datepicker.dateTablePrompt')"
    cellspacing="0"
    cellpadding="0"
    :class="[ns.b(), { 'is-week-mode': selectionMode === 'week' }]"
    @click="handlePickDate"
    @mousemove="handleMouseMove"
  >
    <tbody ref="tbodyRef">
      <tr>
        <th v-if="showWeekNumber" scope="col">{{ t('el.datepicker.week') }}</th>
        <th
          v-for="(week, key) in WEEKS"
          :key="key"
          scope="col"
          :aria-label="t('el.datepicker.weeksFull.' + week)"
        >
          {{ t('el.datepicker.weeks.' + week) }}
        </th>
      </tr>
      <tr
        v-for="(row, key) in rows"
        :key="key"
        :class="[ns.e('row'), { current: isWeekActive(row[1]) }]"
      >
        <td
          v-for="(cell, key_) in row"
          :key="key_"
          :ref="(el) => isSelectedCell(cell) && (currentCellRef = el)"
          :class="getCellClasses(cell)"
          :aria-current="cell.isCurrent ? 'date' : undefined"
          :aria-selected="`${cell.isCurrent}`"
          :tabindex="isSelectedCell(cell) ? 0 : -1"
          @focus="handleFocus"
        >
          <el-date-picker-cell :cell="cell" />
        </td>
      </tr>
    </tbody>
  </table>
</template>

<script lang="ts">
import { computed, defineComponent, nextTick, ref, watch } from 'vue'
import dayjs from 'dayjs'
import { useLocale, useNamespace } from '@element-plus/hooks'
import { castArray } from '@element-plus/utils'
import { basicDateTableProps } from '../props/basic-date-table'
import ElDatePickerCell from './basic-cell-render'

import type { Dayjs } from 'dayjs'
import type { DateCell } from '../date-picker.type'

export default defineComponent({
  components: {
    ElDatePickerCell,
  },
  props: basicDateTableProps,
  emits: ['changerange', 'pick', 'select'],
  expose: ['focus'],
  setup(props, ctx) {
    const ns = useNamespace('date-table')

    const { t, lang } = useLocale()

    const tbodyRef = ref<HTMLElement>()
    const currentCellRef = ref<HTMLElement>()
    // data
    const lastRow = ref(null)
    const lastColumn = ref(null)
    const tableRows = ref<DateCell[][]>([[], [], [], [], [], []])

    // todo better way to get Day.js locale object
    const firstDayOfWeek = (props.date as any).$locale().weekStart || 7
    const WEEKS_CONSTANT = props.date
      .locale('en')
      .localeData()
      .weekdaysShort()
      .map((_) => _.toLowerCase())

    const offsetDay = computed(() => {
      // Sunday 7(0), cal the left and right offset days, 3217654, such as Monday is -1, the is to adjust the position of the first two rows of dates
      return firstDayOfWeek > 3 ? 7 - firstDayOfWeek : -firstDayOfWeek
    })

    const startDate = computed(() => {
      const startDayOfMonth = props.date.startOf('month')
      return startDayOfMonth.subtract(startDayOfMonth.day() || 7, 'day')
    })

    const WEEKS = computed(() => {
      return WEEKS_CONSTANT.concat(WEEKS_CONSTANT).slice(
        firstDayOfWeek,
        firstDayOfWeek + 7
      )
    })

    const hasCurrent = computed<boolean>(() => {
      return rows.value.flat().some((row) => {
        return row.isCurrent
      })
    })

    const rows = computed(() => {
      // TODO: refactory rows / getCellClasses
      const startOfMonth = props.date.startOf('month')
      const startOfMonthDay = startOfMonth.day() || 7 // day of first day
      const dateCountOfMonth = startOfMonth.daysInMonth()
      const dateCountOfLastMonth = startOfMonth
        .subtract(1, 'month')
        .daysInMonth()

      const offset = offsetDay.value
      const rows_ = tableRows.value
      let count = 1

      const selectedDate: Dayjs[] =
        props.selectionMode === 'dates' ? castArray(props.parsedValue) : []

      const calNow = dayjs().locale(lang.value).startOf('day')

      for (let i = 0; i < 6; i++) {
        const row = rows_[i]

        if (props.showWeekNumber) {
          if (!row[0]) {
            row[0] = {
              type: 'week',
              text: startDate.value.add(i * 7 + 1, 'day').week(),
            }
          }
        }

        for (let j = 0; j < 7; j++) {
          let cell = row[props.showWeekNumber ? j + 1 : j]
          if (!cell) {
            cell = {
              row: i,
              column: j,
              type: 'normal',
              inRange: false,
              start: false,
              end: false,
            }
          }
          const index = i * 7 + j
          const calTime = startDate.value.add(index - offset, 'day')
          cell.dayjs = calTime
          cell.date = calTime.toDate()
          cell.timestamp = calTime.valueOf()
          cell.type = 'normal'

          const calEndDate =
            props.rangeState.endDate ||
            props.maxDate ||
            (props.rangeState.selecting && props.minDate)

          cell.inRange =
            (props.minDate &&
              calTime.isSameOrAfter(props.minDate, 'day') &&
              calEndDate &&
              calTime.isSameOrBefore(calEndDate, 'day')) ||
            (props.minDate &&
              calTime.isSameOrBefore(props.minDate, 'day') &&
              calEndDate &&
              calTime.isSameOrAfter(calEndDate, 'day'))

          if (props.minDate?.isSameOrAfter(calEndDate)) {
            cell.start = calEndDate && calTime.isSame(calEndDate, 'day')
            cell.end = props.minDate && calTime.isSame(props.minDate, 'day')
          } else {
            cell.start = props.minDate && calTime.isSame(props.minDate, 'day')
            cell.end = calEndDate && calTime.isSame(calEndDate, 'day')
          }

          const isToday = calTime.isSame(calNow, 'day')

          if (isToday) {
            cell.type = 'today'
          }

          if (i >= 0 && i <= 1) {
            const numberOfDaysFromPreviousMonth =
              startOfMonthDay + offset < 0
                ? 7 + startOfMonthDay + offset
                : startOfMonthDay + offset

            if (j + i * 7 >= numberOfDaysFromPreviousMonth) {
              cell.text = count++
            } else {
              cell.text =
                dateCountOfLastMonth -
                (numberOfDaysFromPreviousMonth - (j % 7)) +
                1 +
                i * 7
              cell.type = 'prev-month'
            }
          } else {
            if (count <= dateCountOfMonth) {
              cell.text = count++
            } else {
              cell.text = count++ - dateCountOfMonth
              cell.type = 'next-month'
            }
          }

          const cellDate = calTime.toDate()
          cell.selected = selectedDate.find(
            (_) => _.valueOf() === calTime.valueOf()
          )
          cell.isSelected = !!cell.selected
          cell.isCurrent = isCurrent(cell)
          cell.disabled = props.disabledDate && props.disabledDate(cellDate)
          cell.customClass =
            props.cellClassName && props.cellClassName(cellDate)
          row[props.showWeekNumber ? j + 1 : j] = cell
        }

        if (props.selectionMode === 'week') {
          const start = props.showWeekNumber ? 1 : 0
          const end = props.showWeekNumber ? 7 : 6
          const isActive = isWeekActive(row[start + 1])
          row[start].inRange = isActive
          row[start].start = isActive
          row[end].inRange = isActive
          row[end].end = isActive
        }
      }
      return rows_
    })

    watch(
      () => props.date,
      async () => {
        if (tbodyRef.value?.contains(document.activeElement)) {
          await nextTick()
          currentCellRef.value?.focus()
        }
      }
    )

    const focus = async () => {
      currentCellRef.value?.focus()
    }

    const isCurrent = (cell): boolean => {
      return (
        props.selectionMode === 'date' &&
        (cell.type === 'normal' || cell.type === 'today') &&
        cellMatchesDate(cell, props.parsedValue)
      )
    }

    const cellMatchesDate = (cell, date) => {
      if (!date) return false
      return dayjs(date)
        .locale(lang.value)
        .isSame(props.date.date(Number(cell.text)), 'day')
    }

    const getCellClasses = (cell) => {
      const classes: string[] = []
      if ((cell.type === 'normal' || cell.type === 'today') && !cell.disabled) {
        classes.push('available')
        if (cell.type === 'today') {
          classes.push('today')
        }
      } else {
        classes.push(cell.type)
      }

      if (isCurrent(cell)) {
        classes.push('current')
      }

      if (
        cell.inRange &&
        (cell.type === 'normal' ||
          cell.type === 'today' ||
          props.selectionMode === 'week')
      ) {
        classes.push('in-range')

        if (cell.start) {
          classes.push('start-date')
        }

        if (cell.end) {
          classes.push('end-date')
        }
      }

      if (cell.disabled) {
        classes.push('disabled')
      }

      if (cell.selected) {
        classes.push('selected')
      }

      if (cell.customClass) {
        classes.push(cell.customClass)
      }

      return classes.join(' ')
    }

    const getDateOfCell = (row, column) => {
      const offsetFromStart =
        row * 7 + (column - (props.showWeekNumber ? 1 : 0)) - offsetDay.value
      return startDate.value.add(offsetFromStart, 'day')
    }

    const handleMouseMove = (event) => {
      if (!props.rangeState.selecting) return

      let target = event.target
      if (target.tagName === 'SPAN') {
        target = target.parentNode.parentNode
      }
      if (target.tagName === 'DIV') {
        target = target.parentNode
      }
      if (target.tagName !== 'TD') return

      const row = target.parentNode.rowIndex - 1
      const column = target.cellIndex

      // can not select disabled date
      if (rows.value[row][column].disabled) return

      // only update rangeState when mouse moves to a new cell
      // this avoids frequent Date object creation and improves performance
      if (row !== lastRow.value || column !== lastColumn.value) {
        lastRow.value = row
        lastColumn.value = column
        ctx.emit('changerange', {
          selecting: true,
          endDate: getDateOfCell(row, column),
        })
      }
    }

    const isSelectedCell = (cell: DateCell) => {
      return (
        (!hasCurrent.value && cell?.text === 1 && cell.type === 'normal') ||
        cell.isCurrent
      )
    }

    const handleFocus = (event: Event) => {
      if (!hasCurrent.value && props.selectionMode === 'date') {
        handlePickDate(event, true)
      }
    }

    const handlePickDate = (event: Event, isKeyboardMovement = false) => {
      let target = event.target as HTMLElement

      target = target?.closest('td')

      if (!target || target.tagName !== 'TD') return

      const row = (target.parentNode as HTMLTableRowElement).rowIndex - 1
      const column = (target as HTMLTableCellElement).cellIndex
      const cell = rows.value[row][column]

      if (cell.disabled || cell.type === 'week') return

      const newDate = getDateOfCell(row, column)

      if (props.selectionMode === 'range') {
        if (!props.rangeState.selecting || !props.minDate) {
          ctx.emit('pick', { minDate: newDate, maxDate: null })
          ctx.emit('select', true)
        } else {
          if (newDate >= props.minDate) {
            ctx.emit('pick', { minDate: props.minDate, maxDate: newDate })
          } else {
            ctx.emit('pick', { minDate: newDate, maxDate: props.minDate })
          }
          ctx.emit('select', false)
        }
      } else if (props.selectionMode === 'date') {
        ctx.emit('pick', newDate, isKeyboardMovement)
      } else if (props.selectionMode === 'week') {
        const weekNumber = newDate.week()
        const value = `${newDate.year()}w${weekNumber}`
        ctx.emit('pick', {
          year: newDate.year(),
          week: weekNumber,
          value,
          date: newDate.startOf('week'),
        })
      } else if (props.selectionMode === 'dates') {
        const newValue = cell.selected
          ? castArray(props.parsedValue).filter(
              (_) => _.valueOf() !== newDate.valueOf()
            )
          : castArray(props.parsedValue).concat([newDate])
        ctx.emit('pick', newValue)
      }
    }

    const isWeekActive = (cell) => {
      if (props.selectionMode !== 'week') return false
      let newDate = props.date.startOf('day')

      if (cell.type === 'prev-month') {
        newDate = newDate.subtract(1, 'month')
      }

      if (cell.type === 'next-month') {
        newDate = newDate.add(1, 'month')
      }

      newDate = newDate.date(Number.parseInt(cell.text, 10))

      if (props.parsedValue && !Array.isArray(props.parsedValue)) {
        const dayOffset =
          ((props.parsedValue.day() - firstDayOfWeek + 7) % 7) - 1
        const weekDate = props.parsedValue.subtract(dayOffset, 'day')
        return weekDate.isSame(newDate, 'day')
      }
      return false
    }

    return {
      ns,
      tbodyRef,
      currentCellRef,
      handleMouseMove,
      t,
      hasCurrent,
      rows,
      isSelectedCell,
      isWeekActive,
      getCellClasses,
      WEEKS,
      handleFocus,
      handlePickDate,
      focus,
    }
  },
})
</script>
