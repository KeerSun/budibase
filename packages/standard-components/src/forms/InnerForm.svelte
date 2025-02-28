<script>
  import { setContext, getContext, onMount } from "svelte"
  import { writable, get } from "svelte/store"
  import { createValidatorFromConstraints } from "./validation"
  import { generateID } from "../helpers"

  export let dataSource
  export let disabled = false
  export let initialValues

  const component = getContext("component")
  const { styleable, API, Provider, ActionTypes } = getContext("sdk")

  let loaded = false
  let schema
  let table
  let fieldMap = {}

  // Form state contains observable data about the form
  const formState = writable({ values: initialValues, errors: {}, valid: true })

  // Form API contains functions to control the form
  const formApi = {
    registerField: (
      field,
      defaultValue = null,
      fieldDisabled = false,
      validationRules
    ) => {
      if (!field) {
        return
      }

      // Auto columns are always disabled
      const isAutoColumn = !!schema?.[field]?.autocolumn

      // Create validation function based on field schema
      const schemaConstraints = schema?.[field]?.constraints
      const validator = createValidatorFromConstraints(
        schemaConstraints,
        validationRules,
        field,
        table
      )

      // Construct field object
      fieldMap[field] = {
        fieldState: makeFieldState(
          field,
          validator,
          defaultValue,
          disabled || fieldDisabled || isAutoColumn
        ),
        fieldApi: makeFieldApi(field, defaultValue),
        fieldSchema: schema?.[field] ?? {},
      }

      // Set initial value
      const initialValue = get(fieldMap[field].fieldState).value
      formState.update(state => ({
        ...state,
        values: {
          ...state.values,
          [field]: initialValue,
        },
      }))

      return fieldMap[field]
    },
    validate: () => {
      const fields = Object.keys(fieldMap)
      fields.forEach(field => {
        const { fieldApi } = fieldMap[field]
        fieldApi.validate()
      })
      return get(formState).valid
    },
    clear: () => {
      const fields = Object.keys(fieldMap)
      fields.forEach(field => {
        const { fieldApi } = fieldMap[field]
        fieldApi.clearValue()
      })
    },
  }

  // Provide both form API and state to children
  setContext("form", { formApi, formState, dataSource })

  // Action context to pass to children
  const actions = [
    { type: ActionTypes.ValidateForm, callback: formApi.validate },
    { type: ActionTypes.ClearForm, callback: formApi.clear },
  ]

  // Creates an API for a specific field
  const makeFieldApi = field => {
    // Sets the value for a certain field and invokes validation
    const setValue = (value, skipCheck = false) => {
      const { fieldState } = fieldMap[field]
      const { validator } = get(fieldState)

      // Skip if the value is the same
      if (!skipCheck && get(fieldState).value === value) {
        return
      }

      // Update field state
      const error = validator ? validator(value) : null
      fieldState.update(state => {
        state.value = value
        state.error = error
        return state
      })

      // Update form state
      formState.update(state => {
        state.values = { ...state.values, [field]: value }
        if (error) {
          state.errors = { ...state.errors, [field]: error }
        } else {
          delete state.errors[field]
        }
        state.valid = Object.keys(state.errors).length === 0
        return state
      })

      return !error
    }

    // Clears the value of a certain field back to the initial value
    const clearValue = () => {
      const { fieldState } = fieldMap[field]
      const { defaultValue } = get(fieldState)
      const newValue = initialValues[field] ?? defaultValue

      // Update field state
      fieldState.update(state => {
        state.value = newValue
        state.error = null
        return state
      })

      // Update form state
      formState.update(state => {
        state.values = { ...state.values, [field]: newValue }
        delete state.errors[field]
        state.valid = Object.keys(state.errors).length === 0
        return state
      })
    }

    // Updates the validator rules for a certain field
    const updateValidation = validationRules => {
      const { fieldState } = fieldMap[field]
      const { value, error } = get(fieldState)

      // Create new validator
      const schemaConstraints = schema?.[field]?.constraints
      const validator = createValidatorFromConstraints(
        schemaConstraints,
        validationRules,
        field,
        table
      )

      // Update validator
      fieldState.update(state => {
        state.validator = validator
        return state
      })

      // If there is currently an error, run the validator again in case
      // the error should be cleared by the new validation rules
      if (error) {
        setValue(value, true)
      }
    }

    return {
      setValue,
      clearValue,
      updateValidation,
      validate: () => {
        const { fieldState } = fieldMap[field]
        setValue(get(fieldState).value, true)
      },
    }
  }

  // Creates observable state data about a specific field
  const makeFieldState = (field, validator, defaultValue, fieldDisabled) => {
    return writable({
      field,
      fieldId: `id-${generateID()}`,
      value: initialValues[field] ?? defaultValue,
      error: null,
      disabled: fieldDisabled,
      defaultValue,
      validator,
    })
  }

  // Fetches the form schema from this form's dataSource, if one exists
  const fetchSchema = async () => {
    if (!dataSource?.tableId) {
      schema = {}
      table = null
    } else {
      table = await API.fetchTableDefinition(dataSource?.tableId)
      if (table) {
        if (dataSource?.type === "query") {
          schema = {}
          const params = table.parameters || []
          params.forEach(param => {
            schema[param.name] = { ...param, type: "string" }
          })
        } else {
          schema = table.schema || {}
        }
      }
    }
    loaded = true
  }

  // Load the form schema on mount
  onMount(fetchSchema)
</script>

<Provider
  {actions}
  data={{ ...$formState.values, tableId: dataSource?.tableId }}
>
  <div use:styleable={$component.styles}>
    {#if loaded}
      <slot />
    {/if}
  </div>
</Provider>
