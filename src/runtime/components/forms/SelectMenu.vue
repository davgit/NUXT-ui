<template>
  <component
    :is="searchable ? 'Combobox' : 'Listbox'"
    v-slot="{ open }"
    :by="by"
    :name="name"
    :model-value="modelValue"
    :multiple="multiple"
    :disabled="disabled"
    as="div"
    :class="ui.wrapper"
    @update:model-value="onUpdate"
  >
    <input
      v-if="required"
      :value="modelValue"
      :required="required"
      class="absolute inset-0 w-px opacity-0 cursor-default"
      tabindex="-1"
      aria-hidden="true"
    >

    <component
      :is="searchable ? 'ComboboxButton' : 'ListboxButton'"
      ref="trigger"
      as="div"
      role="button"
      class="inline-flex w-full"
    >
      <slot :open="open" :disabled="disabled" :loading="loading">
        <button :class="selectMenuClass" :disabled="disabled || loading" type="button">
          <span v-if="(isLeading && leadingIconName) || $slots.leading" :class="leadingWrapperIconClass">
            <slot name="leading" :disabled="disabled" :loading="loading">
              <UIcon :name="leadingIconName" :class="leadingIconClass" />
            </slot>
          </span>

          <slot name="label">
            <span v-if="multiple && Array.isArray(modelValue) && modelValue.length" class="block truncate">{{ modelValue.length }} selected</span>
            <span v-else-if="!multiple && modelValue" class="block truncate">{{ typeof modelValue === 'string' ? modelValue : modelValue[optionAttribute] }}</span>
            <span v-else class="block truncate" :class="ui.placeholder">{{ placeholder || '&nbsp;' }}</span>
          </slot>

          <span v-if="(isTrailing && trailingIconName) || $slots.trailing" :class="trailingWrapperIconClass">
            <slot name="trailing" :disabled="disabled" :loading="loading">
              <UIcon :name="trailingIconName" :class="trailingIconClass" aria-hidden="true" />
            </slot>
          </span>
        </button>
      </slot>
    </component>

    <div v-if="open" ref="container" :class="[ui.container, ui.width]">
      <transition v-bind="ui.transition">
        <component :is="searchable ? 'ComboboxOptions' : 'ListboxOptions'" static :class="[ui.base, ui.divide, ui.ring, ui.rounded, ui.shadow, ui.background, ui.padding, ui.height]">
          <ComboboxInput
            v-if="searchable"
            ref="searchInput"
            :display-value="() => query"
            name="q"
            :placeholder="searchablePlaceholder"
            autofocus
            autocomplete="off"
            :class="ui.input"
            @change="query = $event.target.value"
          />
          <component
            :is="searchable ? 'ComboboxOption' : 'ListboxOption'"
            v-for="(option, index) in filteredOptions"
            v-slot="{ active, selected, disabled: optionDisabled }"
            :key="index"
            as="template"
            :value="option"
            :disabled="option.disabled"
          >
            <li :class="[ui.option.base, ui.option.rounded, ui.option.padding, ui.option.size, ui.option.color, active ? ui.option.active : ui.option.inactive, selected && ui.option.selected, optionDisabled && ui.option.disabled]">
              <div :class="ui.option.container">
                <slot name="option" :option="option" :active="active" :selected="selected">
                  <UIcon v-if="option.icon" :name="option.icon" :class="[ui.option.icon.base, active ? ui.option.icon.active : ui.option.icon.inactive, option.iconClass]" aria-hidden="true" />
                  <UAvatar
                    v-else-if="option.avatar"
                    v-bind="{ size: ui.option.avatar.size, ...option.avatar }"
                    :class="ui.option.avatar.base"
                    aria-hidden="true"
                  />
                  <span v-else-if="option.chip" :class="ui.option.chip.base" :style="{ background: `#${option.chip}` }" />

                  <span class="truncate">{{ typeof option === 'string' ? option : option[optionAttribute] }}</span>
                </slot>
              </div>

              <span v-if="selected" :class="[ui.option.selectedIcon.wrapper, ui.option.selectedIcon.padding]">
                <UIcon :name="selectedIcon" :class="ui.option.selectedIcon.base" aria-hidden="true" />
              </span>
            </li>
          </component>

          <component :is="searchable ? 'ComboboxOption' : 'ListboxOption'" v-if="creatable && queryOption && !filteredOptions.length" v-slot="{ active, selected }" :value="queryOption" as="template">
            <li :class="[ui.option.base, ui.option.rounded, ui.option.padding, ui.option.size, ui.option.color, active ? ui.option.active : ui.option.inactive]">
              <div :class="ui.option.container">
                <slot name="option-create" :option="queryOption" :active="active" :selected="selected">
                  <span class="block truncate">Create "{{ queryOption[optionAttribute] }}"</span>
                </slot>
              </div>
            </li>
          </component>
          <p v-else-if="searchable && query && !filteredOptions.length" :class="ui.option.empty">
            <slot name="option-empty" :query="query">
              No results for "{{ query }}".
            </slot>
          </p>
        </component>
      </transition>
    </div>
  </component>
</template>

<script lang="ts">
import { ref, computed, watch, defineComponent } from 'vue'
import type { PropType, ComponentPublicInstance } from 'vue'
import { defu } from 'defu'
import { Combobox, ComboboxButton, ComboboxOptions, ComboboxOption, ComboboxInput, Listbox, ListboxButton, ListboxOptions, ListboxOption } from '@headlessui/vue'
import UIcon from '../elements/Icon.vue'
import UAvatar from '../elements/Avatar.vue'
import { classNames } from '../../utils'
import { usePopper } from '../../composables/usePopper'
import type { PopperOptions } from '../../types'
import { useAppConfig } from '#imports'
// TODO: Remove
// @ts-expect-error
import appConfig from '#build/app.config'

// const appConfig = useAppConfig()

export default defineComponent({
  components: {
    Combobox,
    ComboboxButton,
    ComboboxOptions,
    ComboboxOption,
    ComboboxInput,
    Listbox,
    ListboxButton,
    ListboxOptions,
    ListboxOption,
    UIcon,
    UAvatar
  },
  props: {
    modelValue: {
      type: [String, Number, Object, Array],
      default: ''
    },
    by: {
      type: String,
      default: undefined
    },
    options: {
      type: Array as PropType<{ [key: string]: any, disabled?: boolean }[] | string[]>,
      default: () => []
    },
    name: {
      type: String,
      default: null
    },
    required: {
      type: Boolean,
      default: false
    },
    icon: {
      type: String,
      default: null
    },
    loadingIcon: {
      type: String,
      default: () => appConfig.ui.input.default.loadingIcon
    },
    leadingIcon: {
      type: String,
      default: null
    },
    trailingIcon: {
      type: String,
      default: () => appConfig.ui.select.default.trailingIcon
    },
    trailing: {
      type: Boolean,
      default: false
    },
    leading: {
      type: Boolean,
      default: false
    },
    loading: {
      type: Boolean,
      default: false
    },
    selectedIcon: {
      type: String,
      default: () => appConfig.ui.selectMenu.default.selectedIcon
    },
    disabled: {
      type: Boolean,
      default: false
    },
    multiple: {
      type: Boolean,
      default: false
    },
    searchable: {
      type: Boolean,
      default: false
    },
    searchablePlaceholder: {
      type: String,
      default: 'Search...'
    },
    creatable: {
      type: Boolean,
      default: false
    },
    placeholder: {
      type: String,
      default: null
    },
    padded: {
      type: Boolean,
      default: true
    },
    size: {
      type: String,
      default: () => appConfig.ui.select.default.size,
      validator (value: string) {
        return Object.keys(appConfig.ui.select.size).includes(value)
      }
    },
    color: {
      type: String,
      default: () => appConfig.ui.select.default.color,
      validator (value: string) {
        return [...appConfig.ui.colors, ...Object.keys(appConfig.ui.select.color)].includes(value)
      }
    },
    variant: {
      type: String,
      default: () => appConfig.ui.select.default.variant,
      validator (value: string) {
        return [
          ...Object.keys(appConfig.ui.select.variant),
          ...Object.values(appConfig.ui.select.color).flatMap(value => Object.keys(value))
        ].includes(value)
      }
    },
    optionAttribute: {
      type: String,
      default: 'label'
    },
    searchAttributes: {
      type: Array,
      default: null
    },
    popper: {
      type: Object as PropType<PopperOptions>,
      default: () => ({})
    },
    ui: {
      type: Object as PropType<Partial<typeof appConfig.ui.selectMenu>>,
      default: () => appConfig.ui.selectMenu
    },
    uiSelect: {
      type: Object as PropType<Partial<typeof appConfig.ui.select>>,
      default: () => appConfig.ui.select
    }
  },
  emits: ['update:modelValue', 'open', 'close'],
  setup (props, { emit, slots }) {
    // TODO: Remove
    const appConfig = useAppConfig()

    const ui = computed<Partial<typeof appConfig.ui.selectMenu>>(() => defu({}, props.ui, appConfig.ui.selectMenu))
    const uiSelect = computed<Partial<typeof appConfig.ui.select>>(() => defu({}, props.uiSelect, appConfig.ui.select))

    const popper = computed<PopperOptions>(() => defu({}, props.popper, ui.value.popper as PopperOptions))

    const [trigger, container] = usePopper(popper.value)

    const query = ref('')
    const searchInput = ref<ComponentPublicInstance<HTMLElement>>()

    const selectMenuClass = computed(() => {
      const variant = uiSelect.value.color?.[props.color as string]?.[props.variant as string] || uiSelect.value.variant[props.variant]

      return classNames(
        uiSelect.value.base,
        uiSelect.value.rounded,
        'text-left cursor-default',
        uiSelect.value.size[props.size],
        uiSelect.value.gap[props.size],
        props.padded && uiSelect.value.padding[props.size],
        variant?.replaceAll('{color}', props.color),
        (isLeading.value || slots.leading) && uiSelect.value.leading.padding[props.size],
        (isTrailing.value || slots.trailing) && uiSelect.value.trailing.padding[props.size],
        uiSelect.value.custom,
        'inline-flex items-center'
      )
    })

    const isLeading = computed(() => {
      return (props.icon && props.leading) || (props.icon && !props.trailing) || (props.loading && !props.trailing) || props.leadingIcon
    })

    const isTrailing = computed(() => {
      return (props.icon && props.trailing) || (props.loading && props.trailing) || props.trailingIcon
    })

    const leadingIconName = computed(() => {
      if (props.loading) {
        return props.loadingIcon
      }

      return props.leadingIcon || props.icon
    })

    const trailingIconName = computed(() => {
      if (props.loading && !isLeading.value) {
        return props.loadingIcon
      }

      return props.trailingIcon || props.icon
    })

    const leadingWrapperIconClass = computed(() => {
      return classNames(
        uiSelect.value.icon.leading.wrapper,
        uiSelect.value.icon.leading.padding[props.size],
        slots.leading && '!pointer-events-auto'
      )
    })

    const leadingIconClass = computed(() => {
      return classNames(
        uiSelect.value.icon.base,
        appConfig.ui.colors.includes(props.color) && uiSelect.value.icon.color.replaceAll('{color}', props.color),
        uiSelect.value.icon.size[props.size],
        props.loading && 'animate-spin'
      )
    })

    const trailingWrapperIconClass = computed(() => {
      return classNames(
        uiSelect.value.icon.trailing.wrapper,
        uiSelect.value.icon.trailing.padding[props.size],
        slots.trailing && '!pointer-events-auto'
      )
    })

    const trailingIconClass = computed(() => {
      return classNames(
        uiSelect.value.icon.base,
        appConfig.ui.colors.includes(props.color) && uiSelect.value.icon.color.replaceAll('{color}', props.color),
        uiSelect.value.icon.size[props.size],
        props.loading && !isLeading.value && 'animate-spin'
      )
    })

    const filteredOptions = computed(() =>
      query.value === ''
        ? props.options
        : (props.options as any[]).filter((option: any) => {
            return (props.searchAttributes?.length ? props.searchAttributes : [props.optionAttribute]).some((searchAttribute: any) => {
              return typeof option === 'string' ? option.search(new RegExp(query.value, 'i')) !== -1 : (option[searchAttribute] && option[searchAttribute].search(new RegExp(query.value, 'i')) !== -1)
            })
          })
    )

    const queryOption = computed(() => {
      return query.value === '' ? null : { [props.optionAttribute]: query.value }
    })

    watch(container, (value) => {
      if (value) {
        emit('open')
      } else {
        emit('close')
      }
    })

    function onUpdate (event: any) {
      if (query.value && searchInput.value?.$el) {
        query.value = ''
        // explicitly set input text because `ComboboxInput` `displayValue` is not reactive
        searchInput.value.$el.value = ''
      }
      emit('update:modelValue', event)
    }

    return {
      // eslint-disable-next-line vue/no-dupe-keys
      ui,
      trigger,
      container,
      isLeading,
      isTrailing,
      selectMenuClass,
      leadingIconName,
      leadingIconClass,
      leadingWrapperIconClass,
      trailingIconName,
      trailingIconClass,
      trailingWrapperIconClass,
      filteredOptions,
      queryOption,
      query,
      onUpdate
    }
  }
})
</script>
