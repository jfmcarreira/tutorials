Boas malta,

Venho partilhar a minha experiência de integração do aspirador da Xiaomi (Roborock S5) no Home Assistant.

[size=5]**Introdução**[/size]

Esta integração tem como objetivos permitir:

* Mandar limpar uma zona a partir do HA;
* Configurar várias zonas de uma vez;
* Suportar o envio de comandos por voz através do Google Assistant em vários idiomas.

A integração com o Google Assistant é feita através do IFTTT (podem seguir este [tutorial](https://forum.cpha.pt/t/broadlink-google-home-ifttt-webhooks-ok-google-channel-269-or-ok-google-change-to-my-favourite-channel-or-ok-google-change-to-cartoon-network/253), mas em vez de canais de televisão são zonas da casa).

Neste tutorial é assumido que o aspirador já está integrado no Home Assistant (neste exemplo a `entity_id: vacuum.roborock`).

O método e código utilizado funcionam tanto com o firmware original como com o [Valetudo](https://github.com/Hypfer/Valetudo) e [Valetudo RE](https://github.com/rand256/valetudo).

[size=5]**Script Python**[/size]
Para alcançar os objetivos mencionados acima cria-se um `python_script`:
```
"""
LIST OF ROOMS / LISTA DE DIVISÔES

The first position is the name of the room. 
These names target voice assistant, therefore a dictionary is created
to provide more than one name to each room, and thus support 
for different languages
  
The second position is the entity_id name and valetudo zone id
The third column is the room_id in the oficial firmware or Valetudo RE app

When using original firmware:
In order to find the room id one can use trial and error using the following command:
    `miiocli  vacuum --ip <IP> --token <TOKEN> segment_clean <integer number>`
and check the output in the xiaomi app
To run this command install `python-miio`

Array example:
#    Room Name                                       Room code name    Room id
vaccum_room_list = [                                                   
    (['sala', 'living room'],                       'living_room',     [18]),
    (['cozinha', 'kitchen'],                        'kitchen',         [19]),
]

vaccum_room_list = [                                                   
    (['sala', 'living room'],                       'living_room'   ),
    (['cozinha', 'kitchen'],                        'kitchen'       ),
]
"""

application_name = "valetudo_re" # or "xiaomi" or "valetudo"

vaccum_room_list = [                                          
    (['sala', 'living room'],              'LivingRoom',   [16]),
    (['corredor', 'hallway'],              'Hallway',      [17]),
    (['cozinha', 'kitchen'],               'Kitchen',      [18]),
    (['casa de banho', 'bathroom'],        'Bathroom',     [23]),
    (['quarto', 'bedroom'],                'Bedroom',      [21, 22]),     
    (['escritório', 'office'],             'Office',       [20]),      
    (['quarto do fundo', 'guest bedroom'], 'GuestBedroom', [19])
]

# Get vacuum entity_id (if more than one)
entity_id = data.get("entity_id", 'vacuum.roborock')

# This is the room name as per 'room_alias'
room = data.get("room").lower()

# Number of runs per room
runs = int( data.get("runs", '1') )

# Start with delay
delay = int( data.get("delay", '0') )

vaccum_room_param = []
if room == "switch_based":
    # Run through all room that are vacuum friendly
    for r in vaccum_room_list:
        entity_name = ('input_boolean.vacuum_'+r[1]).lower()
        should_vaccum = ( hass.states.get( entity_name ).state == 'on' )
        if should_vaccum:
            if application_name == "xiaomi" or application_name == "valetudo_re":
                vaccum_room_param.extend( r[2] )
            elif application_name == "valetudo":
                vaccum_room_param.append( r[1] )

else:
    # Single room
    for r in vaccum_room_list:
        if room in r[0]:
            if application_name == "xiaomi":
                for i in range( runs ):
                    vaccum_room_param.extend( r[2] )
            elif application_name == "valetudo_re":
                vaccum_room_param.extend( r[2] )
            elif application_name == "valetudo":
                vaccum_room_param.append( r[1] )


if application_name == "xiaomi":
    # Service call when using the original xiaomi app
    service_data = { "entity_id": entity_id, "command": "app_segment_clean", "params": vaccum_room_param } 
elif application_name == "valetudo_re":
    # Service call when using the valetudo app
    service_data = { "entity_id": entity_id, "command": "segmented_cleanup", "params": { 'segment_ids': vaccum_room_param, 'repeats': runs } } 
elif application_name == "valetudo":
    # Service call when using the valetudo app
    service_data = { "entity_id": entity_id, "command": "zoned_cleanup", "params": { 'zone_ids': vaccum_room_param } } 


hass.services.call('vacuum','send_command', service_data, False)
```

Neste script deve ser configurado a lista de divisões/zones/rooms da vossa casa.
```
vaccum_room_list = [                                                   
    (['sala', 'living room'],                       'LivingRoom',     [18]),
    (['cozinha', 'kitchen'],                        'Kitchen',         [19]),
]

vaccum_room_list = [                                                   
    (['sala', 'living room'],                       'LivingRoom'   ),
    (['cozinha', 'kitchen'],                        'Kitchen'       ),
]
```
1. Array com o nome da divisão (letra minúscula) em vários idiomas. Util para o controlo por voz
2. Nome de código da divisão/zona tudo pegado. Este é nome usado, por exemplo, no Valetudo.
3. Array com o ids das divisões na app oficial da Xiaomi ou Valetudo RE (1 ou mais). Caso usem Valetudo não precisam desta coluna.
    Para obter estes ids com firmware original podem usar este comando em tentativa erro com números de 1 a 32 (no Valetudo RE tá la na webapp):
    `miiocli  vacuum --ip <IP> --token <TOKEN> segment_clean [<id>]`

Devem também configurar a app que utilizam: `application_name = "valetudo" # or "xiaomi"`
Este `python_script` pode ser chamado no HA usando:
```
service: python_script.vacuum_room
service_data:
    room: Cozinha
    runs: 1
```    
Nota: O runs aqui é para permitir aspirar várias vezes a mesma divisão (apenas testado com app da Xiaomi).

[size=5]**Inputs para as divisões**[/size]
   
Após configurar estas divisões criam um `input_boolean` para cada uma delas:
```
input_boolean:
  vacuum_livingroom:
    name: Sala de Estar
    icon: mdi:sofa
  vacuum_kitchen:
    name: Cozinha
    icon: mdi:stove
```
Notem quem o nome destes `input_boolean` devem ser `vacuum_<codename>` sempre em letra minuscula.

[size=5]**Lovelace**[/size]

Com isto já podemos criar uma view no Lovelace para colocar o nosso aspirador e os `input_boolean` para selecionar as zonas a aspirar.
Para isso é necessário o card [custom:vacuum-card](https://github.com/denysdovhan/vacuum-card).
```
- type: custom:vertical-stack-in-card
  cards:
    - type: "custom:vacuum-card"
      actions:
        - icon: "mdi:sofa"
          name: Aspirar Sala
          service: python_script.vacuum_room
          service_data:
            room: Living Room
            runs: 1
        - icon: "mdi:stove"
          name: Aspirar Cozinha
          service: python_script.vacuum_room
          service_data:
            room: Kitchen
            runs: 1
      compact_view: false
      entity: vacuum.roborock
      show_name: true
      show_toolbar: true
      stats:
        cleaning:
          - attribute: currentCleanArea
            subtitle: Área Limpa
            unit: m2
          - attribute: currentCleanTime
            subtitle: Tempo de Limpeza
            unit: minutes
        default:
          - attribute: filter
            subtitle: Filtro
            unit: hours
          - attribute: sideBrush
            subtitle: Escova Lateral
            unit: hours
          - attribute: mainBrush
            subtitle: Escova Principal
            unit: hours
          - attribute: sensor
            subtitle: Sensores
            unit: hours     
    
    - type: conditional
      conditions:
        - entity: sensor.roborock_state
          state: "error"
      card:
        type: markdown
        title: Erro do Aspirador
        content: |
            {{ states.sensor.roborock_state.attributes.Error }}
      
- type: vertical-stack  
  cards:
    - type: entities
      entities:
        - input_boolean.vacuum_livingroom
        - input_boolean.vacuum_kitchen
      state_color: true
      title: Aspirador-Ready (5 máx.)
    - type: horizontal-stack
      cards:
        - aspect_ratio: 12/2
          entity: python_script.vacuum_room
          icon: "mdi:send"
          name: Aspirar Agora
          layout: icon_name_state
          size: 100%
          tap_action:
            action: call-service
            service: python_script.vacuum_room
            service_data:
              room: switch_based
          type: "custom:button-card"
```

Neste código devem editar as `actions` e o nome dos `input_boolean`. Deve existir uma action e um input para cada divisão. Neste exemplo é mostrado apenas duas divisões.

Terá mais ou menos este aspecto no fim (atenção que no meu caso tenho mais divisões):
![lovelace|640x325](upload://1AVokvz2A9BtfD19NlDMcs4jcUu.png) 

Usando este código ao clicar "Aspirar Agora" ele vai aspirar as divisões selecionadas pelos `input_boolean`.

O conditional card é usado para mostrar a mensagem de erro (enviada por MQTT pelo Valetudo) num `sensor`:
```
sensor:
  - platform: mqtt
    name: Roborock State
    state_topic: "valetudo/Roborock/state"
    value_template: "{{ value_json.state }}"
    json_attributes_topic: "valetudo/Roborock/state"
    json_attributes_template: "{{ {'Error message': value_json.error } | tojson }}"
```
Assim sempre que o state for `error` vai aparecer o texto em markdown por baixo do card do aspirador.

[size=5]**Google Assistant / IFTTT**[/size]

Por fim falta integrar com o Google Assistant. Para tal devem criar uma routina no IFTTT (seguir este [tutorial](https://forum.cpha.pt/t/broadlink-google-home-ifttt-webhooks-ok-google-channel-269-or-ok-google-change-to-my-favourite-channel-or-ok-google-change-to-cartoon-network/253)):
![ifttt|570x480, 100%](upload://bhkWtdV9NA3mwFGZUM8ENJzemSW.png) 


Depois disso criam uma automação para apanhar o comando do IFTTT:
```
automation old:
  # If You say "Clean $ ", then Make a web request
  # Body configured in IFTTT: { "action": "vacuum_room", "room":"{{TextField}}" }
  - alias: IFTTT Vacuum Room
    trigger:
      - event_data:
          action: vacuum_room
        event_type: ifttt_webhook_received
        platform: event
    action:
      - data_template:
          room: "{{ trigger.event.data.room }}"
        service: python_script.vacuum_room
```
Atenção ao body do comando no IFTTT:
```
{ "action": "vacuum_room", "room":"{{TextField}}" }
```
Esta automação vai chamar o `python_script` com o nome da divisão (no meu caso em inglês) e aspirar a divisão correspondente.

[size=5]**Código Fonte**[/size]

Podem consultar todo o YAML que uso no meu Github:
 - [package com a integração](https://github.com/jfmcarreira/home-assistant-config/blob/master/packages/packs/roborock.yaml)
 - [python_script](https://github.com/jfmcarreira/home-assistant-config/blob/master/lovelace/dashboard-main/02-vacuum.yaml)
 - [lovelace](https://github.com/jfmcarreira/home-assistant-config/blob/master/python_scripts/vacuum_room.py)
 

[size=5]**Edit**[/size]

1. O cartão do Lovelace deve ser configurado para os atributos correctos, pois varia entre a integração com o miio e o Valetudo. Foi acrescentado um exemplo.
2. Adicionar conditional card juntamente com um sensor para apanhar a mensagem de erro (testado com Valetudo RE).
3. Melhorado o suporte para Valetudo RE e adicionei o entity_id (obrigado @codedmind)
