# Guia para Redimensionamento da Partição Root no OpenWrt (x86)  

## 1. Introdução  
Este guia é para usuários com experiência em computação e administração de sistemas, mas que são novos no OpenWrt. Ele é baseado na documentação oficial do OpenWrt para hardware x86.  

[Link para documentação oficial](https://openwrt.org/docs/guide-user/installation/openwrt_x86)  

### Mudanças na versão atual  
- **Seção de Diagnóstico adicionada** (agradecimento a @efahl pela sugestão!).  
- **Seção de Redimensionamento do Sistema de Arquivos Root atualizada** para seguir as orientações mais recentes.  

---

## 2. Entendendo o Problema  
### A Origem do OpenWrt  
- O OpenWrt foi originalmente projetado para dispositivos fora do ecossistema x86.  
- Não há mídia de instalação ou instalador tradicional.  
- O sistema é instalado copiando uma imagem diretamente para o disco.  
- Isso resulta em um disco de boot de apenas **120 MB**, independentemente do tamanho real do drive.  

### Como Resolver?  
1. **Redimensionar a partição root** (foco deste guia).  
2. **Criar partições adicionais** (não abordado aqui, mas existem outros tutoriais sobre isso).  

---

## 3. Complicações  
O OpenWrt possui **quatro tipos principais de imagens** para instalação em hardware x86:  

| Tipo de Sistema  | Sistema de Arquivos | Nome da Imagem |
|-----------------|-------------------|--------------|
| **UEFI**       | **ext4**           | generic-ext4-combined-efi.img.gz |
| **BIOS**       | **ext4**           | generic-ext4-combined.img.gz |
| **UEFI**       | **SquashFS**       | generic-squashfs-combined-efi.img.gz |
| **BIOS**       | **SquashFS**       | generic-squashfs-combined.img.gz |

### Como isso afeta o processo?  
- **Em sistemas BIOS:** basta redimensionar a partição e o sistema de arquivos.  
- **Em sistemas UEFI:** é necessário atualizar a configuração do GRUB após redimensionar a partição.  

---

## 4. Segurança  
- O processo pode causar **corrupção do sistema de arquivos**.  
- O ideal é fazer isso com o disco desmontado (boot via USB).  
- Porém, na maioria dos casos, funciona mesmo com o sistema ativo.  

---

## 5. Preparação  
### Instalando as ferramentas necessárias  
Execute o seguinte comando no terminal:  
```sh
opkg update && opkg install lsblk fdisk losetup resize2fs
```

---

## 6. Diagnóstico  
Se você não sabe qual imagem foi usada na instalação, siga estes testes:

### **1. Identificar o sistema de arquivos (ext4 vs SquashFS)**
```sh
df -Th
```
- Se `/dev/root` for **ext4**, o sistema usa ext4.  
- Se `/dev/root` for **squashfs**, o sistema usa SquashFS.  

### **2. Identificar o tipo de firmware (BIOS vs UEFI)**
```sh
ls /sys/firmware/efi
```
- Se a pasta `/sys/firmware/efi` existir, o sistema é **UEFI**.  
- Se der erro (`No such file or directory`), o sistema é **BIOS**.  

---

## 7. Capturar a Configuração Atual das Partições (Somente para UEFI)  
Se seu sistema for **BIOS**, pule para a próxima seção.  

```sh
lsblk -o PATH,SIZE,PARTUUID
```
- Salve essa saída para referência futura.  
- Se estiver via SSH, copie e salve em um arquivo no seu PC.  
- Se estiver com teclado e monitor, salve no dispositivo:  
```sh
lsblk -o PATH,SIZE,PARTUUID > /root/lsblk_old.txt
```
- Para garantir, copie o arquivo para um **pendrive**.  

Verifique também o arquivo de configuração do GRUB:  
```sh
cat /boot/grub/grub.cfg
```
- Procure por referências a `PARTUUID`, pois elas precisarão ser atualizadas.  

---

## 8. Redimensionar a Partição Root  
**O procedimento é o mesmo para BIOS e UEFI.**  

1. Execute `fdisk` no disco:  
```sh
fdisk /dev/sda
```
2. No terminal interativo do `fdisk`, siga os passos:  
   - **Exibir partições:** `p`  
   - **Deletar a partição root:** `d` (escolha a partição correta, normalmente `2`)  
   - **Criar uma nova partição:** `n → p → 2`  
   - **Usar o mesmo setor inicial da partição deletada** (veja o valor com `p`)  
   - **Aceitar o setor final padrão** (máximo possível).  
   - **Manter a assinatura SquashFS, se perguntado:** `n`  
   - **Salvar as alterações:** `w`  

3. Verifique se a partição foi redimensionada:  
```sh
fdisk -l
```

---

## 9. Atualizar a Configuração do GRUB (Somente para UEFI)  
Se o sistema for **BIOS**, pule para a próxima seção.  

1. Obtenha o novo `PARTUUID`:  
```sh
lsblk -o PATH,SIZE,PARTUUID
```
2. Edite o arquivo do GRUB para atualizar o `PARTUUID`:  
```sh
vi /boot/grub/grub.cfg
```
3. Substitua o antigo `PARTUUID` pelo novo, em todas as ocorrências.  

---

## 10. Redimensionar o Sistema de Arquivos  
Agora, precisamos expandir o sistema de arquivos para ocupar o espaço novo.  

### **Se for um sistema ext4**  
```sh
BOOT="$(sed -n -e "\|\s/boot\s.*$|{s///p;q}" /etc/mtab)"
PART="${BOOT##*[^0-9]}"
DISK="${BOOT%${PART}}"
ROOT="${DISK}$((PART+1))"
LOOP="$(losetup -f)"
losetup ${LOOP} ${ROOT}
resize2fs -f ${LOOP}
reboot
```

### **Se for um sistema SquashFS**  
```sh
BOOT="$(sed -n -e "\|\s/boot\s.*$|{s///p;q}" /etc/mtab)"
PART="${BOOT##*[^0-9]}"
DISK="${BOOT%${PART}}"
ROOT="${DISK}$((PART+1))"
LOOP="$(losetup -n -l | sed -n -e "\|\s.*\s${ROOT#/dev}\s.*$|{s///p;q}")"
resize2fs -f ${LOOP}
reboot
```

---

## 11. Conclusão  
- Se tudo funcionou, parabéns!  
- **Reinicie o dispositivo** para garantir que as mudanças tenham sido aplicadas corretamente.  
- Agora, sua partição root ocupa todo o espaço disponível no disco.
