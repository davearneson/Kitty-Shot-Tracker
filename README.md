# Kitty Shot Tracker

This project creates a sensor to track whether my wife or I (or anyone else) have given our diabetic cat his shot twice a day. We were constantly yelling across the house to check if the other person had given the shot because we were so worried about doubling up his dose. (Very dangerous). I realize this is a highly specific case use, but almost every project I’ve ever done of this nature has been born out of seeing someone else’s barely related or seemingly unrelated project and being inspired or enabled through their experience to create something that works for me. I hope this helps someone in that way. I have no programming experience, so I would not be surprised if programming best practices are not in play here, and for that I ask your forgiveness.

## Home Assistant Helpers

I used HA’s helpers to set up three input_boolean entities. I created these in the UI by going to Configuration/Helpers and clicking the add button, then “Toggle.” I then added a name, in my case “Kitty Shot Husband”. I repeated this process twice more and named the input_boolean entities “Kitty Shot Wife” and “Kitty Shot Someone”. These entities will be used as input components in my final sensor. 

# configuration.yaml

There are two sections that have to be added to configuration.yaml for this tracker:

## Template Sensors

There are a couple of things going on here. First of all, there is the kitty_shot template which is outputting text based on the state of the input_boolean entities that we created in the last step. I will create automations later that will not allow any of the input_boolean entities to be on at the same time, but this is the main component of the sensor. 

```
# configuration.yaml

##########Kitty Shot Sensors##########
sensor:
- platform: template
  sensors:
    kitty_shot:
        value_template: >
            {% if is_state('input_boolean.kitty_shot_husband', 'on') %}
                Husband gave kitty his shot.
            {% elif is_state('input_boolean.kitty_shot_wife', 'on') %}
                Wife gave kitty his shot.
            {% elif is_state('input_boolean.kitty_shot_someone', 'on') %}
                Someone gave kitty his shot.
            {% else %}
                Kitty needs his shot.
            {% endif %}
```

Next, this is a template sensor to be used as a component to check to see who gave the shot, and notify the other person that it has been given. I will get more into this later.

```
# configuration.yaml

- platform: template
  sensors:
    kitty_shot_notify:
        value_template: >
            {% if is_state('input_boolean.kitty_shot_husband', 'on') %}
                mobile_app_wife_iphone
            {% elif is_state('input_boolean.kitty_shot_wife', 'on') %}
                mobile_app_husband_iphone
            {% else %}
                Null
            {% endif %}
```

## Actionable Notifications

This part of the project requires that you have set up actionable notifications. I had some trouble doing this when I first did it for another project, but it’s fairly well documented here: https://companion.home-assistant.io/docs/notifications/actionable-notifications/

```
# configuration.yaml

ios:
  push:
    categories:
      - name: Kitty Shot
        identifier: 'kittyshot'
        actions:
          - identifier: 'WIFE_SHOT'
            title: 'Wife shot kitty.'
          - identifier: 'HUSBAND_SHOT'
            title: 'Husband shot kitty.'
          - identifier: 'REMIND_SHOT'
            title: 'Remind me in an hour.'
```
# Automations

I created nine automations for this project, and I’ll do my best to explain them and the logic behind them:

## Shot Needed Notifications

The first few automations send the actionable notification at prescribed times, which vary slightly between weekdays and weekends. (Don’t judge us wanting an extra hour of sleep on weekends!)

```
- alias: Kitty Shot Tracker - Push Notification - Weekdays
  description: ''
  trigger:
  - at: 07:30:00
    platform: time
  - at: '19:30:00'
    platform: time
  condition:
  - condition: time
    weekday:
    - mon
    - tue
    - wed
    - thu
    - fri
  - condition: state
    entity_id: sensor.kitty_shot
    state: Kitty needs his shot.
  action:
  - data:
      data:
        push:
          category: kittyshot
      message: Have you given kitty his shot?
      title: Time for kitty's shot!
    service: notify.mobile_app_husband_iphone
  - data:
      data:
        push:
          category: kittyshot
      message: Have you given kitty his shot?
      title: Time for kitty's shot!
    service: notify.mobile_app_wife_iphone

- alias: Kitty Shot Tracker - Push Notification - Weekends
  description: ''
  trigger:
  - at: 08:30:00
    platform: time
  - at: '19:30:00'
    platform: time
  condition:
  - condition: time
    weekday:
    - sat
    - sun
  - condition: state
    entity_id: sensor.kitty_shot
    state: Kitty needs his shot.
  action:
  - data:
      data:
        push:
          category: kittyshot
      message: Have you given kitty his shot?
      title: Time for kitty's shot!
    service: notify.mobile_app_husband_iphone
  - data:
      data:
        push:
          category: kittyshot
      message: Have you given kitty his shot?
      title: Time for kitty's shot!
    service: notify.mobile_app_wife_iphone
```

## Other Automation Components

This next set is pulling triple duty. The automation is run if the boolean switch is turned on manually (through the card UI) or if the actionable notification is fired. These automations also prohibit any of the three switches from being on at the same time.

```
- alias: Kitty Shot Tracker - iOS Response - Wife
  description: ''
  trigger:
  - entity_id: input_boolean.kitty_shot_wife
    platform: state
    to: 'on'
  - event_data:
      actionName: WIFE_SHOT
    event_type: ios.notification_action_fired
    platform: event
  condition: []
  action:
  - data: {}
    entity_id: input_boolean.kitty_shot_husband
    service: input_boolean.turn_off
  - data: {}
    entity_id: input_boolean.kitty_shot_wife
    service: input_boolean.turn_on
  - data: {}
    entity_id: input_boolean.kitty_shot_someone
    service: input_boolean.turn_off

- alias: Kitty Shot Tracker - iOS Response - Husband
  description: ''
  trigger:
  - event_data:
      actionName: HUSBAND_SHOT
    event_type: ios.notification_action_fired
    platform: event
  - entity_id: input_boolean.kitty_shot_husband
    platform: state
    to: 'on'
  condition: []
  action:
  - data: {}
    entity_id: input_boolean.kitty_shot_husband
    service: input_boolean.turn_on
  - data: {}
    entity_id: input_boolean.kitty_shot_wife
    service: input_boolean.turn_off
  - data: {}
    entity_id: input_boolean.kitty_shot_someone
    service: input_boolean.turn_off
```

This automation sends the reminder notification again in one hour. This is useful if no one is home or available to give the shot at the moment, and we need to be reminded later. I decided not to do an arriving home trigger in case we were already home, but unable to get to it right that moment.

```
- alias: Kitty Shot Tracker - iOS Response - Remind me in an hour
  description: ''
  trigger:
  - event_data:
      actionName: REMIND_SHOT
    event_type: ios.notification_action_fired
    platform: event
  condition: []
  action:
  - delay: 01:00:00
  - data: {}
    entity_id: automation.kitty_shot_tracker_push_notification_weekdays
    service: automation.trigger
```

I set up this automation to use with an Amazon Echo Button that we had lying around. If one of us gives the cat his shot while we don’t have our phone handy, we can just push the echo button which is linked to the input_boolean.kitty_shot_someone entity through an Alexa routine. Because it could be either of us (or a cat sitter), we settled on the terminology “Someone.” Both of us will be alerted if this button is pushed. 

```
- alias: Kitty Shot Tracker - Notify that shot has been given by "SOMEONE"
  description: ''
  trigger:
  - entity_id: sensor.kitty_shot
    from: Kitty needs his shot.
    platform: state
    to: Someone gave kitty his shot.
  condition: []
  action:
  - data_template:
      message: '{{ states(''sensor.kitty_shot'') }}'
      title: Kitty has been shot!
    service: notify.mobile_app_husband_iphone
  - data_template:
      message: '{{ states(''sensor.kitty_shot'') }}'
      title: Kitty has been shot!
    service: notify.mobile_app_wife_iphone
```

## Shot Given Notifications

This automation uses the kitty_shot_notify sensor I set up at the beginning in configuration.yaml to check and see who gave the cat his shot, and notify the other person so we don’t double up.

```
- alias: Kitty Shot Tracker - Notify that shot has been given
  description: ''
  trigger:
  - entity_id: sensor.kitty_shot
    from: Kitty needs his shot.
    platform: state
  condition: []
  action:
  - data_template:
      message: '{{ states(''sensor.kitty_shot'') }}'
      title: Kitty has been shot!
    service_template: notify.{{ states.sensor.kitty_shot_notify.state }}
```

This automation prevents getting duplicate notifications once the shot has been given. 

```
- alias: Kitty Shot Tracker - Automation Component
  description: Adds logic to not send notification if changing who gave shot.
  trigger:
  - entity_id: input_boolean.kitty_shot_someone
    platform: state
    to: 'on'
  condition:
  - condition: or
    conditions:
    - condition: state
      entity_id: input_boolean.kitty_shot_husband
      state: 'on'
    - condition: state
      entity_id: input_boolean.kitty_shot_wife
      state: 'on'
  action:
  - data: {}
    entity_id: input_boolean.kitty_shot_husband
    service: input_boolean.turn_off
  - data: {}
    entity_id: input_boolean.kitty_shot_wife
    service: input_boolean.turn_off
  - data: {}
    entity_id: input_boolean.kitty_shot_someone
    service: input_boolean.turn_on
```

## Reset Tracker

Finally, this automation resets the tracker twice a day, so when it’s time to give the shot again, everything is registered as off.

```
- alias: Kitty Shot Tracker - Reset Tracker
  description: ''
  trigger:
  - at: 06:00:00
    platform: time
  - at: '17:30:00'
    platform: time
  condition: []
  action:
  - data: {}
    entity_id: input_boolean.kitty_shot_husband
    service: input_boolean.turn_off
  - data: {}
    entity_id: input_boolean.kitty_shot_wife
    service: input_boolean.turn_off
  - data: {}
    entity_id: input_boolean.kitty_shot_someone
    service: input_boolean.turn_off
```

# Custom Card for Lovelace UI

I combined everything into a custom picture elements card. I had originally used an entity card with three button cards combined into stacks, but found I preferred the look of everything unified together, so I came up with this:

```
elements:
  - entity: sensor.kitty_shot
    style:
      bottom: 58%
      color: black
      font-size: 24px
      left: 3.2%
      transform: initial
    type: state-label
  - entity: input_boolean.kitty_shot_wife
    state_image:
      'off': 'https://i.imgur.com/FDSN008.png'
      'on': 'https://i.imgur.com/FDSN008.png'
    style:
      left: 91%
      top: 25.5%
      width: 18.2%
    tap_action:
      action: none
    type: image
  - entity: input_boolean.kitty_shot_wife
    state_image:
      'off': 'https://i.imgur.com/HjzKYy0.png'
      'on': 'https://i.imgur.com/HjzKYy0.png'
    style:
      left: 17%
      top: 73.5%
      width: 31.2%
    tap_action:
      action: call-service
      service: input_boolean.turn_on
      service_data:
        entity_id: input_boolean.kitty_shot_wife
    type: image
  - entity: input_boolean.kitty_shot_wife
    icon: 'mdi:needle'
    style:
      left: 13%
      top: 57%
      transform: 'scale(2.8,2.8)'
    tap_action:
      action: call-service
      service: input_boolean.turn_on
      service_data:
        entity_id: input_boolean.kitty_shot_wife
    type: state-icon
  - entity: input_boolean.kitty_shot_husband
    state_image:
      'off': 'https://i.imgur.com/7wszxUu.png'
      'on': 'https://i.imgur.com/7wszxUu.png'
    style:
      left: 50%
      top: 73.5%
      width: 31.2%
    tap_action:
      action: call-service
      service: input_boolean.turn_on
      service_data:
        entity_id: input_boolean.kitty_shot_husband
    type: image
  - entity: input_boolean.kitty_shot_husband
    icon: 'mdi:needle'
    style:
      left: 46%
      top: 57%
      transform: 'scale(2.8,2.8)'
    tap_action:
      action: call-service
      service: input_boolean.turn_on
      service_data:
        entity_id: input_boolean.kitty_shot_husband
    type: state-icon
  - entity: input_boolean.kitty_shot_someone
    state_image:
      'off': 'https://i.imgur.com/4jHEQ28.png'
      'on': 'https://i.imgur.com/4jHEQ28.png'
    style:
      left: 83%
      top: 73.5%
      width: 31.2%
    tap_action:
      action: call-service
      service: input_boolean.turn_off
      service_data:
        entity_id:
          - input_boolean.kitty_shot_husband
          - input_boolean.kitty_shot_wife
          - input_boolean.kitty_shot_someone
    type: image
image: 'https://i.imgur.com/vxmYbYb.png'
type: picture-elements
```

