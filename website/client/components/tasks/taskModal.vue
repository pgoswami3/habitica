<template lang="pug">
form(
  v-if="task",
  @submit.stop.prevent="submit()",
)
  b-modal#task-modal(
    size="sm",
    @hidden="cancel()",
  )
    .task-modal-header(
      slot="modal-header",
      :class="[cssClass]",
    )
      h1 {{ title }}
      .form-group
        label(v-once) {{ `${$t('title')}*` }}
        input.form-control(type='text', :class="[`${cssClass}-modal-input`]", required, v-model="task.text")
      .form-group
        label(v-once) {{ $t('notes') }}
        textarea.form-control(:class="[`${cssClass}-modal-input`]", v-model="task.notes", rows="3")
    .task-modal-content
      .option(v-if="task.type === 'reward'")
        label(v-once) {{ $t('cost') }}
        input(type="number", v-model="task.value", required, min="0")
      .option(v-if="['daily', 'todo'].indexOf(task.type) > -1")
        label(v-once) {{ $t('checklist') }}
        br
        .checklist-group.input-group(v-for="(item, $index) in task.checklist")
          input.checklist-item.form-control(type="text", :value="item.text")
          span.input-group-btn(@click="removeChecklistItem($index)")
            .svg-icon.destroy-icon(v-html="icons.destroy")
        input.checklist-item.form-control(type="text", :placeholder="$t('newChecklistItem')", @keydown.enter="addChecklistItem($event)", v-model="newChecklistItem")
      .d-flex.justify-content-center(v-if="task.type === 'habit'")
        .option-item(:class="{'option-item-selected': task.up === true}", @click="task.up = !task.up")
          .option-item-box
            .task-control.habit-control(:class="controlClass.up + '-control-habit'")
              .svg-icon.positive(v-html="icons.positive")
          .option-item-label(v-once) {{ $t('positive') }}
        .option-item(:class="{'option-item-selected': task.down === true}", @click="task.down = !task.down")
          .option-item-box
            .task-control.habit-control(:class="controlClass.down + '-control-habit'")
              .svg-icon.negative(v-html="icons.negative")
          .option-item-label(v-once) {{ $t('negative') }}
      template(v-if="task.type !== 'reward'")
        label(v-once)
          span.float-left {{ $t('difficulty') }}
          .svg-icon.info-icon(v-html="icons.information")
        .d-flex.justify-content-center
          .option-item(:class="{'option-item-selected': task.priority === 0.1}", @click="task.priority = 0.1")
            .option-item-box
              .svg-icon.difficulty-trivial-icon(v-html="icons.difficultyTrivial")
            .option-item-label(v-once) {{ $t('trivial') }}
          .option-item(:class="{'option-item-selected': task.priority === 1}", @click="task.priority = 1")
            .option-item-box
              .svg-icon.difficulty-normal-icon(v-html="icons.difficultyNormal")
            .option-item-label(v-once) {{ $t('easy') }}
          .option-item(:class="{'option-item-selected': task.priority === 1.5}", @click="task.priority = 1.5")
            .option-item-box
              .svg-icon.difficulty-medium-icon(v-html="icons.difficultyMedium")
            .option-item-label(v-once) {{ $t('medium') }}
          .option-item(:class="{'option-item-selected': task.priority === 2}", @click="task.priority = 2")
            .option-item-box
              .svg-icon.difficulty-hard-icon(v-html="icons.difficultyHard")
            .option-item-label(v-once) {{ $t('hard') }}
      .option(v-if="task.type === 'todo'")
        label(v-once) {{ $t('dueDate') }}
        datepicker(v-model="task.date")
      .option(v-if="task.type === 'daily'")
        label(v-once) {{ $t('startDate') }}
        datepicker(v-model="task.startDate")
      .option(v-if="task.type === 'daily'")
        label(v-once) {{ $t('repeats') }}
        b-dropdown(:text="$t(task.frequency)")
          b-dropdown-item(v-for="frequency in ['daily', 'weekly', 'monthly', 'yearly']", :key="frequency", @click="task.frequency = frequency", :class="{active: task.frequency === frequency}")
            | {{ $t(frequency) }}
        label(v-once) {{ $t('repeatEvery') }}
        input.form-control(type="number", v-model="task.everyX", min="0", required)
        | {{ repeatSuffix }}
        template(v-if="task.frequency === 'weekly'")
          .form-check(
            v-for="(day, dayNumber) in dayMapping",
            :key="dayNumber",
          )
            label.custom-control.custom-checkbox
              input.custom-control-input(type="checkbox", v-model="task.repeat[day]")
              span.custom-control-indicator
              span.custom-control-description(v-once) {{ weekdaysMin(dayNumber) }}
        template(v-if="task.frequency === 'monthly'")
          label.custom-control.custom-radio
            input.custom-control-input(type='radio', v-model="repeatsOn", value="dayOfMonth")
            span.custom-control-indicator
            span.custom-control-description {{ $t('dayOfMonth') }}
          label.custom-control.custom-radio
            input.custom-control-input(type='radio', v-model="repeatsOn", value="dayOfWeek")
            span.custom-control-indicator
            span.custom-control-description {{ $t('dayOfWeek') }}

      .option
        label(v-once) {{ $t('tags') }}
        .category-wrap(@click="showTagsSelect = !showTagsSelect")
          span.category-select(v-if='task.tags && task.tags.length === 0') {{$t('none')}}
          span.category-select(v-else) {{getTagsFor(task)[0]}}
        .category-box(v-if="showTagsSelect")
          .form-check(
            v-for="tag in user.tags",
            :key="tag.id",
          )
            label.custom-control.custom-checkbox
              input.custom-control-input(type="checkbox", :value="tag.id", v-model="task.tags")
              span.custom-control-indicator
              span.custom-control-description(v-once) {{ tag.name }}
          button.btn.btn-primary(@click="showTagsSelect = !showTagsSelect") {{$t('close')}}
      .option(v-if="task.type === 'habit'")
        label(v-once) {{ $t('resetStreak') }}
        b-dropdown(:text="$t(task.frequency)")
          b-dropdown-item(v-for="frequency in ['daily', 'weekly', 'monthly']", :key="frequency", @click="task.frequency = frequency", :class="{active: task.frequency === frequency}")
            | {{ $t(frequency) }}

    .task-modal-footer(slot="modal-footer")
      button.btn.btn-primary(type="submit", v-once) {{ $t('save') }}
      span.cancel-task-btn(v-once, v-if="purpose === 'create'", @click="cancel()") {{ $t('cancel') }}
      span.delete-task-btn(v-once, v-else, @click="destroy()") {{ $t('delete') }}
</template>

<style lang="scss">
  @import '~client/assets/scss/colors.scss';

  #task-modal {
    .modal-dialog.modal-sm {
      max-width: 448px;
    }

    label {
      font-weight: bold;
    }

    input, textarea {
      border: none;
      background-color: rgba(0, 0, 0, 0.16);

      &:focus {
        color: $white !important;
      }
    }

    .modal-content {
      border-radius: 8px;
      border: none;
    }

    .modal-header, .modal-body, .modal-footer {
      padding: 0px;
      border: none;
    }

    .task-modal-content, .task-modal-header {
      padding-left: 23px;
      padding-right: 23px;
    }

    .task-modal-header {
      color: $white;
      width: 100%;
      border-top-left-radius: 8px;
      border-top-right-radius: 8px;
      padding-top: 16px;
      padding-bottom: 24px;

      h1 {
        color: $white;
      }
    }

    .task-modal-content {
      padding-top: 24px;

      input {
        background: $white;
        border: 1px solid $gray-500;
        color: $gray-200;

        &:focus {
          color: $gray-50 !important;
        }
      }
    }

    .info-icon {
      float: left;
      height: 16px;
      width: 16px;
      margin-left: 8px;
      margin-top: 2px;
    }

    .difficulty-trivial-icon {
      width: 16px;
      height: 16px;
    }

    .difficulty-normal-icon {
      width: 36px;
      height: 16px;
    }

    .difficulty-medium-icon {
      width: 36px;
      height: 32px;
    }

    .difficulty-hard-icon {
      width: 36px;
      height: 36px;
    }

    .option {
      margin-bottom: 12px;
      margin-top: 12px;
      position: relative;
    }

    .option-item {
      margin-right: 48px;
      cursor: pointer;

      &:last-child {
        margin-right: 0px;
      }

      &-box {
        width: 64px;
        height: 64px;
        border-radius: 2px;
        background: $gray-600;
        margin-bottom: 8px;
        display: flex;
        align-items: center;
        justify-content: center;

        .habit-control.task-habit-disabled-control-habit {
          color: $white !important;
          border: none;
          background: $gray-300;
        }
      }

      &-label {
        color: $gray-50;
        text-align: center;
      }
    }

    .category-wrap {
      cursor: pointer;
      margin-top: 0px;
    }

    .category-box {
      bottom: 0px;
      left: 40px;
      top: auto;
    }

    .checklist-group {
      border-top: 1px solid $gray-500;

      .input-group-btn {
        cursor: pointer;
        padding-left: 10px;
        padding-right: 10px;
      }

      .destroy-icon {
        width: 14px;
        height: 16px;
      }
    }

    .checklist-item {
      margin-bottom: 0px;
      border-radius: 0px;
      border: none !important;
      padding-left: 36px;

      &:last-child {
        background-size: 10px 10px;
        background-image: url(~client/assets/svg/for-css/positive.svg);
        background-repeat: no-repeat;
        background-position: center left 10px;
        border-top: 1px solid $gray-500 !important;
        border-bottom: 1px solid $gray-500 !important;
      }
    }

    .task-modal-footer {
      margin: 0 auto;
      padding-bottom: 24px;
      border-top-left-radius: 8px;
      border-top-right-radius: 8px;
      margin-top: 50px;

      .delete-task-btn, .cancel-task-btn {
        margin-left: 16px;
        cursor: pointer;

        &:hover, &:focus, &:active {
          text-decoration: underline;
        }
      }

      .delete-task-btn {
        color: $red-50;
      }

      .cancel-task-btn {
        color: $blue-10;
      }
    }
  }
</style>

<script>
import bModal from 'bootstrap-vue/lib/components/modal';
import { mapGetters, mapActions, mapState } from 'client/libs/store';
import bDropdown from 'bootstrap-vue/lib/components/dropdown';
import bDropdownItem from 'bootstrap-vue/lib/components/dropdown-item';
import Datepicker from 'vuejs-datepicker';
import moment from 'moment';

import informationIcon from 'assets/svg/information.svg';
import difficultyTrivialIcon from 'assets/svg/difficulty-trivial.svg';
import difficultyMediumIcon from 'assets/svg/difficulty-medium.svg';
import difficultyHardIcon from 'assets/svg/difficulty-hard.svg';
import difficultyNormalIcon from 'assets/svg/difficulty-normal.svg';
import positiveIcon from 'assets/svg/positive.svg';
import negativeIcon from 'assets/svg/negative.svg';
import deleteIcon from 'assets/svg/delete.svg';

export default {
  components: {
    bModal,
    bDropdown,
    bDropdownItem,
    Datepicker,
  },
  props: ['task', 'purpose', 'challengeId'], // purpose is either create or edit, task is the task created or edited
  data () {
    return {
      showTagsSelect: false,
      newChecklistItem: null,
      icons: Object.freeze({
        information: informationIcon,
        difficultyNormal: difficultyNormalIcon,
        difficultyTrivial: difficultyTrivialIcon,
        difficultyMedium: difficultyMediumIcon,
        difficultyHard: difficultyHardIcon,
        negative: negativeIcon,
        positive: positiveIcon,
        destroy: deleteIcon,
      }),
    };
  },
  computed: {
    ...mapGetters({
      getTaskClasses: 'tasks:getTaskClasses',
      getTagsFor: 'tasks:getTagsFor',
    }),
    ...mapState({
      user: 'user.data',
      dayMapping: 'constants.DAY_MAPPING',
    }),
    title () {
      const type = this.$t(this.task.type);
      return this.$t(this.purpose === 'edit' ? 'editATask' : 'createTask', {type});
    },
    cssClass () {
      return this.getTaskClasses(this.task, this.purpose === 'edit' ? 'editModal' : 'createModal');
    },
    controlClass () {
      return this.getTaskClasses(this.task, this.purpose === 'edit' ? 'control' : 'controlCreate');
    },
    repeatSuffix () {
      const task = this.task;

      if (task.frequency === 'daily') {
        return task.everyX === 1 ? this.$t('day') : this.$t('days');
      } else if (task.frequency === 'weekly') {
        return task.everyX === 1 ? this.$t('week') : this.$t('weeks');
      } else if (task.frequency === 'monthly') {
        return task.everyX === 1 ? this.$t('month') : this.$t('months');
      } else if (task.frequency === 'yearly') {
        return task.everyX === 1 ? this.$t('year') : this.$t('years');
      }
    },
    repeatsOn: {
      get () {
        let repeatsOn = 'dayOfMonth';

        if (this.task.type === 'daily' && this.task.weeksOfMonth && this.task.weeksOfMonth.length > 0) {
          repeatsOn = 'dayOfWeek';
        }

        return repeatsOn;
      },
      set (newValue) {
        const task = this.task;

        if (task.frequency === 'monthly' && newValue === 'dayOfMonth') {
          const date = moment(task.startDate).date();
          task.weeksOfMonth = [];
          task.daysOfMonth = [date];
        } else if (task.frequency === 'monthly' && newValue === 'dayOfWeek') {
          const week = Math.ceil(moment(task.startDate).date() / 7) - 1;
          const dayOfWeek = moment(task.startDate).day();
          const shortDay = this.dayMapping[dayOfWeek];
          task.daysOfMonth = [];
          task.weeksOfMonth = [week];
          for (let key in task.repeat) {
            task.repeat[key] = false;
          }
          task.repeat[shortDay] = true;
        }
      },
    },
  },
  methods: {
    ...mapActions({saveTask: 'tasks:save', destroyTask: 'tasks:destroy', createTask: 'tasks:create'}),
    addChecklistItem (e) {
      this.task.checklist.push({text: this.newChecklistItem, completed: false});
      this.newChecklistItem = null;
      e.preventDefault();
    },
    removeChecklistItem (i) {
      this.task.checklist.splice(i, 1);
    },
    weekdaysMin (dayNumber) {
      return moment.weekdaysMin(dayNumber);
    },
    submit () {
      if (this.purpose === 'create') {
        if (this.challengeId) {
          this.$store.dispatch('tasks:createChallengeTasks', {
            challengeId: this.challengeId,
            tasks: [this.task],
          });
          this.$emit('taskCreated', this.task);
        } else {
          this.createTask(this.task);
        }
      } else {
        this.saveTask(this.task);
        this.$emit('taskEdited', this.task);
      }
      this.$root.$emit('hide::modal', 'task-modal');
    },
    destroy () {
      this.destroyTask(this.task);
      this.$root.$emit('hide::modal', 'task-modal');
    },
    cancel () {
      this.showTagsSelect = false;
      this.$emit('cancel');
    },
  },
};
</script>
