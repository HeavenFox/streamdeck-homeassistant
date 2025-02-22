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

axios.defaults.headers = {
  'Cache-Control': 'public, max-age=86400'
}

let entityConfigFactory
const svgUtils = new SvgUtils()

const $SD = ref(null)
const $HA = ref(null)
const $reconnectTimeout = ref({})
const globalSettings = ref({})
const actionSettings = ref([])
const buttonLongpressTimeouts = ref(new Map()) //context, timeout

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
        entityConfigFactory = new EntityConfigFactory(inGlobalSettings.displayConfiguration?.urlOverride || inGlobalSettings.displayConfiguration?.url)
        connectHomeAssistant()
      }
    )

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

      if (rotationTimeout[context])
        return

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
    axios.get('https://cdn.jsdelivr.net/gh/cgiesche/streamdeck-homeassistant@master/public/config/active-states.yml')
      .then(response => activeStates.value = yaml.load(response.data))
      .catch(error => console.log(`Failed to download updated active-states.json: ${error}`))
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
    $HA.value = new Homeassistant(globalSettings.value.serverUrl, globalSettings.value.accessToken, onHAConnected, onHAError, onHAClosed)
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
  Object.keys(actionSettings.value).forEach(key => $SD.value.showAlert(key))
}

function entityStatesChanged(event) {
  event.forEach(updateState)
}

function entityStateChanged(event) {
  if (event) {
    let newState = event.data.new_state
    updateState(newState)
  }
}

function updateState(stateMessage) {
  if (!stateMessage.entity_id) {
    console.log(`Missing entity_id in updated state: ${stateMessage}`)
    return
  }

  let domain = stateMessage.entity_id.split('.')[0]
  let changedContexts = Object.keys(actionSettings.value).filter(key => actionSettings.value[key].display.entityId === stateMessage.entity_id)

  changedContexts.forEach(context => {
    try {
      if (stateMessage.last_updated != null) stateMessage.attributes['last_updated'] = new Date(stateMessage.last_updated).toLocaleTimeString()
      if (stateMessage.last_changed != null) stateMessage.attributes['last_changed'] = new Date(stateMessage.last_changed).toLocaleTimeString()

      updateContextState(context, domain, stateMessage)
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

function updateContextState(currentContext, domain, stateObject) {
  let contextSettings = actionSettings.value[currentContext]
  let renderingConfig = entityConfigFactory.determineConfig(domain, stateObject, contextSettings.display)

  renderingConfig.isAction = contextSettings.button.serviceShortPress.serviceId && (contextSettings.display.enableServiceIndicator === undefined || contextSettings.display.enableServiceIndicator) // undefined = on by default
  renderingConfig.isMultiAction = contextSettings.button.serviceLongPress.serviceId && (contextSettings.display.enableServiceIndicator === undefined || contextSettings.display.enableServiceIndicator) // undefined = on by default

  if (renderingConfig.rotationPercent !== undefined) {
    rotationPercent[currentContext] = renderingConfig.rotationPercent
  }

  if (contextSettings.display.useCustomTitle) {
    let state = stateObject.state
    let stateAttributes = stateObject.attributes
    renderingConfig.customTitle = nunjucks.renderString(contextSettings.display.buttonTitle, { ...{ state }, ...stateAttributes })
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
    renderingConfig.feedback.icon = 'data:image/svg+xml;charset=utf8,' + svgUtils.renderIconSVG(renderingConfig.icon, renderingConfig.color)
    if (renderingConfig.feedback.value === undefined) {
      renderingConfig.feedback.value = svgUtils.renderTemplates(renderingConfig.labelTemplates, { ...stateObject.attributes, ...{ state: stateObject.state } }).join(' ')
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

    renderingConfig.isAction = contextSettings.button.serviceShortPress.serviceId && (contextSettings.display.enableServiceIndicator === undefined || contextSettings.display.enableServiceIndicator) // undefined = on by default
    renderingConfig.isMultiAction = contextSettings.button.serviceLongPress.serviceId && (contextSettings.display.enableServiceIndicator === undefined || contextSettings.display.enableServiceIndicator) // undefined = on by default

    if (!renderingConfig.color) {
      renderingConfig.color = activeStates.value.indexOf(stateObject.state) !== -1 ? entityConfigFactory.colors.active : entityConfigFactory.colors.neutral
    }

    const buttonSVG = svgUtils.renderButtonSVG(renderingConfig, stateObject)
    setButtonSVG(buttonSVG, currentContext)
  }
}

function setButtonSVG(svg, changedContext) {
  const image = 'data:image/svg+xml;,' + svg
  if (actionSettings.value[changedContext].controllerType === 'Encoder') {
    $SD.value.setFeedbackLayout(changedContext, { 'layout': '$A0' })
    $SD.value.setFeedback(changedContext, { 'full-canvas': image, 'canvas': null, 'title': '' })
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
          let renderedServiceData = nunjucks.renderString(serviceToCall.serviceData, serviceDataAttributes)
          serviceData = JSON.parse(renderedServiceData)
        }

        $HA.value.callService(serviceIdParts[1], serviceIdParts[0], serviceToCall.entityId, serviceData)
      } catch (e) {
        console.error(e)
        $SD.value.showAlert(context)
      }
    }
  }
}

</script>
