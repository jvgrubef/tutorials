O tutorial oferece um guia passo a passo para configurar minha máquina virtual OPNSense no Proxmox. Desde a configuração inicial da VM até ajustes finos para otimização de desempenho, o tutorial cobre todos os aspectos essenciais que preciso considerar. Ele aborda a seleção de configurações de hardware, instalação do sistema operacional OPNSense, configuração de redes e discos, e ajustes detalhados na interface web do OPNSense e no sistema operacional para garantir uma conexão estável e baixa latência. É uma referência valiosa para mim, já que necessito configurar minha VM OPNSense de forma eficiente e otimizada.
Ver. Promxox: 8.1
Ver. OPNSense: 24.1

## Passo 1 - Criação de VM OPNSense no Proxmox

### Geral
Defina ID, Nome e marque inicialização no boot.  

### SO
Selecione a ISO do OPNSense.  
Selecione o tipo "Other" (importante).  

### Sistema
Placa de vídeo: padrão.  
Máquina: q35.  
BIOS: OVMF (UEFI).  
Adicione o armazenamento do EFI.  
Desmarque a caixa "Pre-Enroll keys" (importante).  
Controlador: Virtio SCIS Single (importante).  
Marque o agente QEMU.  
Marque o TPM e adicione o armazenamento, deixe na versão v2.0 (Opcional).  

### Discos
Barramento selecione VirtIO Block.  
No cache, selecione Writeback.  
Marque IO Thread.  
Desmarque o backup (Opcional).  
No async IO selecione IO_Uring.  

### CPU
Socket: 1.  
CPUs: O máximo de threads do servidor.  
Tipo de CPU: Host (importante).  
vCPUs: teste o desempenho entre 3 e 4 (importante).  
Unidades de CPU: 2048 ou mais (importante).  
Extras:  
- Caso tenha processador seja Westmere, Sandbridge ou Ivybridge, marque "pcid".  
- Caso seu servidor suporte above 4g, marque "pdpe1gb".  
- Marque AES (importante).  

### Memória
De pelo menos 2 GB de memória, recomendado são 4 GB.  
Desmarque dispositivo Ballooning.  

### Rede
Selecione a ponte que será a WAN.  
Desmarque o firewall.  
Marque desconectado.  
Defina o MAC caso:  
- Seu ISP use seu MAC para permitir a conexão (muito comum em PPPOE).  
- Se precisar de um MAC específico para essa interface.  

### Confirme
Não marque para inicializar ainda.  

## Passo 2 - Vá para a aba de hardware
Adicione mais interfaces de rede, quantas precisar usando o mesmo método anterior, mas desta vez coloque a ponte que será a lan, guest, etc; Mas seja minimalista, quanto menos interfaces, mais rápido e estável será.  
Qualquer configuração de bridge, faca diretamente no Proxmox;  
Adicione um Virtio RNG e selecione /dev/urandom ou /dev/hwrng se disponível (opcional).  

## Passo 3 - Vá para a aba opções
Desmarque o ponteiro de tablet.  
Confira a ordem do boot para a imagem de disco do OPNSense.  
Selecione apenas rede em hotplug.  
Selecione "Não" no Local para RTC.  

## Passo 4 - Inicie a VM e instale o OPNSense
Preste atenção, quando pedir para definir as interfaces antes do sistema inicializar por completo, faça uma configuração básica:  
LAGG: N  
VLAN: N  
WAN: vtnet0  
LAN: vtnet1  
Deixe o próximo item vazio e aperte enter para salvar.  

Quando aparecer a tela de login, entre com "installer" e a senha é opnsense, continue com default key maps, selecione partição UFS e use o disco Virt block, após instalação, opcionalmente você pode definir a senha, em seguida, reinicie e preste atenção, deixando o mouse sobre a VM na lista lateral, quando a VM reiniciar na BIOS, pare a VM à força (Stop), vá na aba hardware, nas interfaces de rede, desmarque a opção desconectado, remova o disco ISO do OPNSense, vá na aba opções, e na ordem do boot, marque apenas o virtio e confirme, depois inicie a VM.  

## Passo 4.5 - Primeiro login
Faça login no webui do OPNSense, faça os primeiros passos e configure as conexões, atualize o sistema e reinicie. Após reinício, faça login no webui novamente e instale o pacote do agente do convidado QEMU e reinicie.  

## Passo 5 - Ajustes finos
Faça login no webui do OPNSense, depois vá em System/Settings/Tunables e modifique caso existam ou adicione as seguintes opções:  

```
hw.vtnet.lro_disable       = 1  
hw.vtnet.tso_disable       = 1  
hw.vtnet.csum_disable      = 1  
hw.pci.honor_msi_blacklist = 0  
net.inet.rss.enabled       = 1  
net.inet.rss.bits          = 2  
net.isr.maxthreads         = -1  
```

Depois, reinicie. Agora na VM, faça login e use o (8) shell. Com o comando ping, use algum dispositivo em sua rede para executar o teste de latência (ex: um roteador que vai ser usado como AP ou um computador na rede). Verifique se a latência ainda fica abaixo de 1ms enquanto executa testes de velocidade. Caso ainda tenha muitos picos extremos, você pode tentar adicionar as seguintes configurações ao seu OPNSense também, mas não as recomendo por motivos de segurança:  

```
hw.ibrs_disable            = 1  
vm.pmap.pti                = 0  
```

Reinicie e faça os testes novamente. Este será o melhor desempenho da sua VM de OPNSense. Note que outras VMs ou containers podem influenciar no desempenho, mas, no geral, você deve conseguir uma conexão sem perda de pacotes e com latência aceitável.  
