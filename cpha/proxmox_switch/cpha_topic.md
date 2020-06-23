Boas malta,

Para quem estiver interessado em controlar as VMs ou LXC containers do Proxmox apartir do Home Assistant fica aqui o meu exemplo.

Para começar deve seguir os passos da [Integração Oficial do Proxmox no Home Assistant](https://www.home-assistant.io/integrations/proxmoxve/).
Para este integração funcionar deve criar um novo `user` e um novo `role` com a permissão `VM.Audit` para permitir obter informação sobre as VMs/LXC.

Para podermos controlar o seu estado devem atribuir também a permissão `VM.PowerMgmt` como podem ver na figura abaixo:
