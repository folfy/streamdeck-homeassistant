<template>
  <span>Nothing to see here!</span>
</template>

<script setup>
import { StreamDeck } from '@/modules/common/streamdeck'
import { Homeassistant } from '@/modules/homeassistant/homeassistant'
import { SvgUtils } from '@/modules/plugin/svgUtils'
import nunjucks from 'nunjucks'
import { Settings } from '@/modules/common/settings'
import { onMounted, ref } from 'vue'
import { EntityConfigFactory } from '@/modules/plugin/entityConfigFactoryNg'
import defaultActiveStates from '../../public/config/active-states.yml'
import axios from 'axios'
import yaml from 'js-yaml'

let entityConfigFactory
const svgUtils = new SvgUtils()

const $SD = ref(null)
const $HA = ref(null)
const $reconnectTimeout = ref({})
const globalSettings = ref({})
const actionSettings = ref([])
const buttonLongpressTimeouts = ref(new Map()) //context, timeout
const latestStates = ref({}) // entityId -> stateMessage

const activeStates = ref(defaultActiveStates)

let rotationTimeout = []
let rotationAmount = []
let rotationPercent = []

onMounted(async () => {
  window.connectElgatoStreamDeckSocket = (inPort, inPluginUUID, inRegisterEvent, inInfo) => {
    $SD.value = new StreamDeck(inPort, inPluginUUID, inRegisterEvent, inInfo, '{}')

    $SD.value.on('globalsettings', (inGlobalSettings) => {
      console.log('Got global settings.')
      globalSettings.value = inGlobalSettings
      entityConfigFactory = new EntityConfigFactory(
        inGlobalSettings.displayConfiguration?.urlOverride ||
          inGlobalSettings.displayConfiguration?.url
      )
      connectHomeAssistant()
    })

    $SD.value.on('connected', () => {
      $SD.value.requestGlobalSettings()
    })

    $SD.value.on('keyDown', (message) => {
      buttonDown(message.context)
    })

    $SD.value.on('keyUp', (message) => {
      buttonUp(message.context)
    })

    $SD.value.on('willAppear', (message) => {
      let context = message.context
      rotationAmount[context] = 0
      rotationPercent[context] = 0
      actionSettings.value[context] = Settings.parse(message.payload.settings)
      if ($HA.value) {
        $HA.value.getStatesDebounced(entityStatesChanged)
      }
    })

    $SD.value.on('willDisappear', (message) => {
      let context = message.context
      delete actionSettings.value[context]
    })

    $SD.value.on('dialRotate', (message) => {
      let context = message.context
      let settings = actionSettings.value[context]
      let scaledTicks = message.payload.ticks * (settings.rotationTickMultiplier || 1)
      let tickBucketSizeMs = settings.rotationTickBucketSizeMs || 300

      rotationAmount[context] += scaledTicks
      rotationPercent[context] += scaledTicks
      if (rotationPercent[context] < 0) {
        rotationPercent[context] = 0
      } else if (rotationPercent[context] > 100) {
        rotationPercent[context] = 100
      }

      if (rotationTimeout[context]) return

      let serviceCall = () => {
        callService(context, settings.button.serviceRotation, {
          ticks: rotationAmount[context],
          rotationPercent: rotationPercent[context],
          rotationAbsolute: 2.55 * rotationPercent[context]
        })
        rotationAmount[context] = 0
        rotationTimeout[context] = null
      }

      if (tickBucketSizeMs > 0) {
        rotationTimeout[context] = setTimeout(serviceCall, tickBucketSizeMs)
      } else {
        serviceCall()
      }
    })

    $SD.value.on('dialDown', (message) => {
      buttonDown(message.context)
    })

    $SD.value.on('dialUp', (message) => {
      buttonUp(message.context)
    })

    $SD.value.on('touchTap', (message) => {
      let context = message.context
      let settings = actionSettings.value[context]
      callService(context, settings.button.serviceTap)
    })

    $SD.value.on('didReceiveSettings', (message) => {
      let context = message.context
      rotationAmount[context] = 0
      actionSettings.value[context] = Settings.parse(message.payload.settings)
      if ($HA.value) {
        $HA.value.getStatesDebounced(entityStatesChanged)
      }
    })
  }

  await fetchActiveStates()
})

async function fetchActiveStates() {
  try {
    axios
      .get(
        'https://cdn.jsdelivr.net/gh/cgiesche/streamdeck-homeassistant@master/public/config/active-states.yml'
      )
      .then((response) => (activeStates.value = yaml.load(response.data)))
      .catch((error) => console.log(`Failed to download updated active-states.json: ${error}`))
  } catch (error) {
    console.log(`Failed to download updated active-states.json: ${error}`)
  }
}

function connectHomeAssistant() {
  console.log('Connecting to Home Assistant')
  if (globalSettings.value.serverUrl && globalSettings.value.accessToken) {
    if ($HA.value) {
      $HA.value.close()
    }
    console.log('Connecting to Home Assistant ' + globalSettings.value.serverUrl)
    $HA.value = new Homeassistant(
      globalSettings.value.serverUrl,
      globalSettings.value.accessToken,
      onHAConnected,
      onHAError,
      onHAClosed
    )
  }
}

const onHAConnected = () => {
  $HA.value.getStatesDebounced(entityStatesChanged)
  $HA.value.subscribeEvents(entityStateChanged)
}

function onHAError(msg) {
  showAlert()
  console.log(`Home Assistant connection error: ${msg}`)
  window.clearTimeout($reconnectTimeout)
  $reconnectTimeout.value = window.setTimeout(connectHomeAssistant, 5000)
}

function onHAClosed(msg) {
  showAlert()
  console.log(`Home Assistant connection closed, trying to reopen connection: ${msg}`)
  window.clearTimeout($reconnectTimeout)
  $reconnectTimeout.value = window.setTimeout(connectHomeAssistant, 5000)
}

function showAlert() {
  Object.keys(actionSettings.value).forEach((key) => $SD.value.showAlert(key))
}

function entityStatesChanged(states) {
  states.forEach((state) => {
    latestStates.value[state.entity_id] = state
  })
  states.forEach(updateState)
}

function entityStateChanged(event) {
  if (event) {
    let newState = event.data.new_state
    latestStates.value[newState.entity_id] = newState
    updateState(newState)
  }
}

function resolveState(entityId, incomingState) {
  return entityId === incomingState.entity_id ? incomingState : latestStates.value[entityId]
}

function updateState(stateMessage) {
  if (!stateMessage.entity_id) {
    console.log(`Missing entity_id in updated state: ${stateMessage}`)
    return
  }

  let changedContexts = Object.keys(actionSettings.value).filter((key) => {
    const display = actionSettings.value[key].display
    return (
      display.entityId === stateMessage.entity_id ||
      (display.additionalEntityIds || []).includes(stateMessage.entity_id)
    )
  })

  changedContexts.forEach((context) => {
    try {
      const contextDisplay = actionSettings.value[context].display

      // Resolve the primary state — use the incoming message if it is for the primary entity,
      // otherwise fall back to the cached state.
      const primaryState = resolveState(contextDisplay.entityId, stateMessage)

      if (!primaryState) {
        // Primary state not yet loaded; skip until a full state refresh arrives.
        return
      }

      if (primaryState.last_updated != null)
        primaryState.attributes['last_updated'] = new Date(
          primaryState.last_updated
        ).toLocaleTimeString()
      if (primaryState.last_changed != null)
        primaryState.attributes['last_changed'] = new Date(
          primaryState.last_changed
        ).toLocaleTimeString()

      const additionalStates = (contextDisplay.additionalEntityIds || [])
        .map((id) => resolveState(id, stateMessage))
        .filter(Boolean)

      const primaryDomain = primaryState.entity_id.split('.')[0]
      updateContextState(context, primaryDomain, primaryState, additionalStates)
    } catch (e) {
      console.error(e)
      $SD.value.setImage(context, null)
      $SD.value.showAlert(context)
    }
  })
}

function isEncoder(contextSettings) {
  return contextSettings.controllerType === 'Encoder'
}

function updateContextState(currentContext, domain, stateObject, additionalStates = []) {
  let contextSettings = actionSettings.value[currentContext]

  // Build the entities array (entities[0] = primary, entities[1..N] = additional) and
  // enrich stateObject.attributes so that all template rendering paths share the same context.
  const entitiesContext = [stateObject, ...additionalStates].map((s) => ({
    state: s.state,
    entity_id: s.entity_id,
    ...s.attributes
  }))
  const enrichedStateObject = {
    ...stateObject,
    attributes: { ...stateObject.attributes, entities: entitiesContext }
  }

  let renderingConfig = entityConfigFactory.determineConfig(
    domain,
    enrichedStateObject,
    contextSettings.display
  )

  renderingConfig.isAction =
    contextSettings.button.serviceShortPress.serviceId &&
    (contextSettings.display.enableServiceIndicator === undefined ||
      contextSettings.display.enableServiceIndicator) // undefined = on by default
  renderingConfig.isMultiAction =
    contextSettings.button.serviceLongPress.serviceId &&
    (contextSettings.display.enableServiceIndicator === undefined ||
      contextSettings.display.enableServiceIndicator) // undefined = on by default

  if (renderingConfig.rotationPercent !== undefined) {
    rotationPercent[currentContext] = renderingConfig.rotationPercent
  }

  if (contextSettings.display.useCustomTitle) {
    let state = enrichedStateObject.state
    let stateAttributes = enrichedStateObject.attributes
    renderingConfig.customTitle = nunjucks.renderString(contextSettings.display.buttonTitle, {
      ...{ state },
      ...stateAttributes
    })
  }

  if (contextSettings.display.useCustomButtonLabels) {
    renderingConfig.labelTemplates = contextSettings.display.buttonLabels.split('\n')
  }

  if (isEncoder(contextSettings)) {
    if (!renderingConfig.feedbackLayout) {
      renderingConfig.feedbackLayout = '$A1'
    }
    $SD.value.setFeedbackLayout(currentContext, { layout: renderingConfig.feedbackLayout })

    if (!renderingConfig.feedback) {
      renderingConfig.feedback = {}
    }
    renderingConfig.feedback.title = renderingConfig.customTitle ? renderingConfig.customTitle : ''
    renderingConfig.feedback.icon =
      'data:image/svg+xml;charset=utf8,' +
      svgUtils.renderIconSVG(renderingConfig.icon, renderingConfig.color)
    if (renderingConfig.feedback.value === undefined) {
      renderingConfig.feedback.value = svgUtils
        .renderTemplates(renderingConfig.labelTemplates, {
          ...enrichedStateObject.attributes,
          ...{ state: enrichedStateObject.state }
        })
        .join(' ')
    }
    $SD.value.setFeedback(currentContext, renderingConfig.feedback)
  } else if (contextSettings.display.useStateImagesForOnOffStates) {
    if (renderingConfig.customTitle) {
      $SD.value.setTitle(currentContext, renderingConfig.customTitle)
    }
    if (activeStates.value.indexOf(stateObject.state) !== -1) {
      console.log('Setting state of ' + currentContext + ' to 1')
      $SD.value.setState(currentContext, 1)
    } else {
      console.log('Setting state of ' + currentContext + ' to 0')
      $SD.value.setState(currentContext, 0)
    }
  } else {
    if (renderingConfig.customTitle) {
      $SD.value.setTitle(currentContext, renderingConfig.customTitle)
    }

    renderingConfig.isAction =
      contextSettings.button.serviceShortPress.serviceId &&
      (contextSettings.display.enableServiceIndicator === undefined ||
        contextSettings.display.enableServiceIndicator) // undefined = on by default
    renderingConfig.isMultiAction =
      contextSettings.button.serviceLongPress.serviceId &&
      (contextSettings.display.enableServiceIndicator === undefined ||
        contextSettings.display.enableServiceIndicator) // undefined = on by default

    if (!renderingConfig.color) {
      renderingConfig.color =
        activeStates.value.indexOf(stateObject.state) !== -1
          ? entityConfigFactory.colors.active
          : entityConfigFactory.colors.neutral
    }

    const buttonSVG = svgUtils.renderButtonSVG(renderingConfig, enrichedStateObject)
    setButtonSVG(buttonSVG, currentContext)
  }
}

function setButtonSVG(svg, changedContext) {
  const image = 'data:image/svg+xml;,' + svg
  if (actionSettings.value[changedContext].controllerType === 'Encoder') {
    $SD.value.setFeedbackLayout(changedContext, { layout: '$A0' })
    $SD.value.setFeedback(changedContext, { 'full-canvas': image, canvas: null, title: '' })
  } else {
    $SD.value.setImage(changedContext, image)
  }
}

function buttonDown(context) {
  const timeout = setTimeout(buttonLongPress, 300, context)
  buttonLongpressTimeouts.value.set(context, timeout)
}

function buttonUp(context) {
  // If "long press timeout" is still present, we perform a normal press
  const lpTimeout = buttonLongpressTimeouts.value.get(context)
  if (lpTimeout) {
    clearTimeout(lpTimeout)
    buttonLongpressTimeouts.value.delete(context)
    buttonShortPress(context)
  }
}

function buttonShortPress(context) {
  let settings = actionSettings.value[context]
  callService(context, settings.button.serviceShortPress)
}

function buttonLongPress(context) {
  buttonLongpressTimeouts.value.delete(context)
  let settings = actionSettings.value[context]
  if (settings.button.serviceLongPress.serviceId) {
    callService(context, settings.button.serviceLongPress)
  } else {
    callService(context, settings.button.serviceShortPress)
  }
}

function callService(context, serviceToCall, serviceDataAttributes = {}) {
  if ($HA.value) {
    if (serviceToCall['serviceId']) {
      try {
        const serviceIdParts = serviceToCall.serviceId.split('.')

        let serviceData = null
        if (serviceToCall.serviceData) {
          let renderedServiceData = nunjucks.renderString(
            serviceToCall.serviceData,
            serviceDataAttributes
          )
          serviceData = JSON.parse(renderedServiceData)
        }

        $HA.value.callService(
          serviceIdParts[1],
          serviceIdParts[0],
          serviceToCall.entityId,
          serviceData
        )
      } catch (e) {
        console.error(e)
        $SD.value.showAlert(context)
      }
    }
  }
}
</script>
