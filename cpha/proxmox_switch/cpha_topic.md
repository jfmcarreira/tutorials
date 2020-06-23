Olá malta,

Para quem estiver interessado em controlar as VMs ou LXC containers do Proxmox apartir do Home Assistant fica aqui o meu exemplo.

Para começar deve seguir os passos da [Integração Oficial do Proxmox no Home Assistant](https://www.home-assistant.io/integrations/proxmoxve/).
Para este integração funcionar deve criar um novo `user` e um novo `role` com a permissão `VM.Audit` para permitir obter informação sobre as VMs/LXC.

Para podermos controlar o seu estado devem atribuir também a permissão `VM.PowerMgmt` como podem ver na figura abaixo:
![proxmox|640x288, 75%](upload://oVprpeClaJXOjNjuHhQKYFkF02o.png) 

Depois disto devem fazer download do seguinte [shell script](https://github.com/jfmcarreira/tutorials/blob/master/cpha/proxmox_switch/proxmox_control_vm.bash) e coloca-lo, por exemplo, na pasta `/config/shell_scripts`.
Nota: caso usem outro directório mudem o caminho quando criarem o `shell_command` usando o código abaixo.
O script tem as seguintes entradas (por esta ordem):
1. IP do Proxmox;
2. Nome do Node, hydrogen no meu caso (não acreditei que fosse preciso um `secret` aqui);
3. Utilizador criado;
4. Palavra-chave;
5. Comando a correr (start/stop)
6. Tipo (vm/lxc);
7. ID do LXC/VM.

Caso usem o `template switch` que apresento apenas precisam de adicionar os `secrets` e mudar os argumentos do `service call`.

Depois disto podem usar o código abaixo para criarem `template switch` para cada uma das VMs/LXC a controlar.
```
sensor:
  - platform: template
    sensors:
      proxmox_ip:
        value_template: !secret proxmox_ip
      proxmox_user:
        value_template: !secret proxmox_user
      proxmox_pass:
        value_template: !secret proxmox_pass
        
        
shell_command:
  proxmox_control_vm: bash /config/shell_scripts/proxmox_control_vm.bash {{ states("sensor.proxmox_ip") }} hydrogen {{ states("sensor.proxmox_user") }} {{ states("sensor.proxmox_pass") }} {{ command }} {{ type }} {{ vm }} 
  
switch:
  - platform: template
    switches:
      proxmox_zoneminder_vm:
        value_template: "{{ is_state('binary_sensor.hydrogen_zoneminder_running', 'on') }}"
        turn_on:
          service: shell_command.proxmox_control_vm
          data:
            type: lxc
            vm: 112
            command: start
        turn_off:
          service: shell_command.proxmox_control_vm
          data:
            type: lxc
            vm: 112
            command: stop
            
  - platform: template
    switches:
      proxmox_francium_vm:
        value_template: "{{ is_state('binary_sensor.hydrogen_francium_running', 'on') }}"
        turn_on:
          service: shell_command.proxmox_control_vm
          data:
            type: qemu
            vm: 300
            command: start
        turn_off:
          service: shell_command.proxmox_control_vm
          data:
            type: qemu
            vm: 300
            command: stop
```

Este código contém um exemplo para um LXC e para uma VM. Devem mudar o `type: vm/lxc` e um número da VM/LXC `vm: 300`.

Este tutorial assume que estão a usar a integração oficial e têm um `binary_sensor` para as VMs a controlar que é usado para definir o estado do `switch`.
Isto é importante para a seguinte configuração ser válida: `value_template: "{{ is_state('binary_sensor.hydrogen_francium_running', 'on') }}"`.

Nota: Não acredito que a forma usada seja a mais eficiente para esconder os dados de autenticação. Caso não partilhem o vosso código podem inserir os dados diretamente no `shell_command`
